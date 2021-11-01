# AMIA Nov 2021 - NLP demo

1. Input data prep
	- Clone synthea repo, and follow the quick start guide
		```bash
		cd <amia demo directory>
		git clone https://github.com/synthetichealth/synthea.git
		mv synthea.diff synthea/
		cd synthea
		git checkout a3482c856d30410e438047845796095a827cca41
		./gradlew build check test
		```
	- Apply the patch to update the settings in `./synthea/src/main/resources/synthea.properties` to allow for creating bulk data and clinical notes 
		```bash
		git apply synthea.diff
		```
		
	- run synthea to create patient data
		`./run_synthea -p 1000` or `./run_synthea -p 1000 -m covid19` (for covid-19-related fake data)

2. Prepare cTAKES server
	- option 1, use demo server {server ip: 3.142.162.98:4000 , try 3.142.162.98:8080 if your firewall blocked the default server}. 
		The server is available Nov 1-3, 2021 for AMIA demo use only.
	- option 2, install you own server in localhost (require ~4G memory and ~12G storage). 
		Register at UMLS (https://www.nlm.nih.gov/research/umls/index.html). Obtain API key after registration.
		```bash
		cd <amia demo directory>
		git clone https://github.com/Machine-Learning-for-Medical-Language/ctakes-covid-container   # Private repo ?
		cd ctakes-covid-container
		docker build -t ctakes-covid .
		export umls_api_key={UMLS-api-key}
		./start_rest.sh
		```
		Confirm the server is running by check localhost:8080 for Tomcat.
		check container logs if localhost:8080 is not accessible:
			i. "umls user valided" message should appear in the logs
			ii. the container takes a while to start, the last message should be similar to "org.apache.catalina.startup.Catalina.start Server startup in [16,375] milliseconds"


3. NLP process
	This process read in clinical notes from ./synthea/output/notes/\*.txt files, send to cTAKES server. The result is converted in to FHIR R4 resources.
	Current version support medical terms in 4 categories, with one-to-one FHIR resource mapping 
  		DiseaseDisorderMention => Condition
  		SignSymptomMention => Observation
  		MedicationMention => MedicationStatement
  		ProcedureMention => Procedure
  	```bash
	cd <amia demo directory>
  	git clone https://github.com/bmao1/NLP_to_FHIR
	mkdir -p NLP_to_FHIR/output/fhir
	cp synthea/output/fhir/* NLP_to_FHIR/output/fhir/
  	cd NLP_to_FHIR
  	python extract_cuis_edits.py ../synthea/output/notes/ output/fhir true
  	```

4. Loading processed structured data into analytics database (postgres)
	- Start postgres container. DB data will be stored in ./data subdirectory; Input data should be placed in ./input subdirectory (Edit volumne mapping as needed)
	```bash
	cd <amia demo directory>
	mkdir db_data
	docker run -d\
		-p 5432:5432 \
		--name postgres13 \
		-e POSTGRES_USER=postgres \
		-e POSTGRES_PASSWORD=example \
		-e POSTGRES_DB=amia \
		-e POSTGRES_HOST=postgres \
		-e PGDATA=/var/lib/postgresql/data/pgdata \
		-v `pwd`/db_data:/var/lib/postgresql/data \
		-v `pwd`/NLP_to_FHIR/output/fhir:/var/lib/postgresql/input \
		--rm \
		postgres:13
	```

	- Using the following command to load data. For demo purpose, all FHIR resources are loaded into one table.
		There is a known issue where text generated by synthea contains `\"`, which causes an error when loading into postgres, so the sed command below is to doubly escape them before loading.
		
		Load data using postgres cli:
		```
		# Start interactive CLI 
		docker exec -it postgres13 psql -U postgres amia

		# Create table 
		CREATE TABLE IF NOT EXISTS demo (contents jsonb);
		
		# Load ndjson file 
		\copy demo from program 'sed -e ''s/\\/\\\\/g'' /var/lib/postgresql/input/Patient.ndjson'
		\copy demo from program 'sed -e ''s/\\/\\\\/g'' /var/lib/postgresql/input/Condition.ndjson'
		\copy demo from program 'sed -e ''s/\\/\\\\/g'' /var/lib/postgresql/input/Procedure.ndjson'
		\copy demo from program 'sed -e ''s/\\/\\\\/g'' /var/lib/postgresql/input/MedicationStatement.ndjson'
		\copy demo from program 'sed -e ''s/\\/\\\\/g'' /var/lib/postgresql/input/Observation.ndjson'
		```
 	

5. Sample query to get patient gender, dob, race, ethnicity. Query can run via CLI or API
```sql
select id
	, gender
	, birthDate
	, max(race)
	, max(ethnicity)
	from (
		select contents -> 'id' as id
			, contents -> 'gender' as gender
			, (contents ->> 'birthDate')::timestamptz as birthDate
			, case when (extext -> 'valueCoding' ->> 'code') in ('1002-5','2028-9','2054-5','2076-8','2106-3') then (extext -> 'valueCoding' ->> 'display') else '' end as race
			, case when (extext -> 'valueCoding' ->> 'code') in ('2135-2','2186-5') then (extext -> 'valueCoding' ->> 'display') else '' end as ethnicity
		from demo, jsonb_array_elements(contents -> 'extension') ext, jsonb_array_elements(ext -> 'extension') extext
		where contents->> 'resourceType' = 'Patient' 
			and extext ->> 'url' = 'ombCategory'
	) as pt
	group by id
		, gender
		, birthDate
;
```
	

6. Sample query to get demographic distribution for loss of taste symptom

```sql
select gender
        , race
        , ethnicity
        , count(distinct case when covid_mention='Yes' then pt.id end) as covid_mention
        , count(distinct case when loss_of_taste='Yes' then pt.id end) as loss_of_taste
        , count(distinct pt.id) as patient_cnt
from (select distinct contents ->> 'id' as id
                , contents ->> 'gender' as gender
                , max(case when (extext -> 'valueCoding' ->> 'code') in ('1002-5','2028-9','2054-5','2076-8','2106-3') then (extext -> 'valueCoding' ->> 'display') else '' end) as race
                , max(case when (extext -> 'valueCoding' ->> 'code') in ('2135-2','2186-5') then (extext -> 'valueCoding' ->> 'display') else '' end) as ethnicity
        from demo, jsonb_array_elements(contents -> 'extension') ext, jsonb_array_elements(ext -> 'extension') extext
        where contents->> 'resourceType' = 'Patient'
        group by id, gender
                ) as pt
        left join
        (select id
                , max(covid_mention) as covid_mention
                , max(loss_of_taste) as loss_of_taste
                from (
                        SELECT RIGHT(contents -> 'subject' ->> 'reference', 36)  as id
                                , case when (coding_arr ->> 'system' = 'urn:oid:2.16.840.1.113883.6.86' and coding_arr ->> 'code' = 'a0_72' ) then 'Yes' end as covid_mention
                                , case when (coding_arr ->> 'system' = 'urn:oid:2.16.840.1.113883.6.86' and coding_arr ->> 'code' = 'C2364111' ) then 'Yes' end as loss_of_taste
                        FROM demo
                                , jsonb_array_elements(contents -> 'code' -> 'coding')  coding_arr
                        where contents->> 'resourceType' = 'Observation'
                        ) as obs
                group by id
                ) as covid
on pt.id = covid.id
group by gender, race, ethnicity;
```




