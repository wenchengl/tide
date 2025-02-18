# TiDE Overview

TiDE is a free text deidentification tool that can identify and deid PHI in clinical note text and other free text in medical data. It uses pattern matching, known PHI matching and NER to search for PHI, and use geenral replacement or hide-in-plain-sight to replace PHI with safe text.


## Safe Harbor 18 identifiers
TiDE can identify the following HIPAA identifiers either by pattern matching or known PHI matching:

```
  Name, Address, dates,phone,fax,Email,SSN,MRN,Health plan beneficiary number,Account number,Certificate or license number,vehicle number,URL,IP, Any other characteristic that could uniquely identify the individual
```

TiDE does not process non-text information such as these two identifiers
```
Finger/Voice print,photo
```

## Optionally use Google DLP API to identify the following DLP infoTypes: 
https://cloud.google.com/dlp/docs/infotypes-reference

```
  AGE,DATE,DATE_OF_BIRTH,CREDIT_CARD_NUMBER,US_BANK_ROUTING_MICR,AMERICAN_BANKERS_CUSIP_ID,IBAN_CODE,US_ADOPTION_TAXPAYER_IDENTIFICATION_NUMBER,US_DRIVERS_LICENSE_NUMBER,US_INDIVIDUAL_TAXPAYER_IDENTIFICATION_NUMBER,US_PREPARER_TAXPAYER_IDENTIFICATION_NUMBER,US_PASSPORT,US_SOCIAL_SECURITY_NUMBER,US_EMPLOYER_IDENTIFICATION_NUMBER,US_VEHICLE_IDENTIFICATION_NUMBER,EMAIL_ADDRESS,PERSON_NAME,PHONE_NUMBER,US_HEALTHCARE_NPI,US_DEA_NUMBER,LOCATION,IP_ADDRESS,MAC_ADDRESS,URL
```


# Data Preparation for TiDE

## Directly use BigQuery table as input 

You can directly use BigQuery table that contains the free text column as input if you only need to use NER or general patten matching. If you have known PHI associated with text, you can join free text with knwon phi, and create a final input table with both text and known phi in each row. 

## Use JSON files on Google Cloud Storage or local disk

If you prefer to work with files, you can export table to Google Cloud Storage bucket as newline delimited JSON files. Then use these GCS files as input.


# Options to Replace PHI 

Options to deid PHI discovered by TiDE:

* Masking (everything except name and location)
* Jittering (date, age) with provided jitter value
* Surrogate name and location
* General Replacement with common patterns of each type of PHIs

# Build the project

Run Maven commands at repo root directory.
```
mvn clean install -DskipTests=true
```

# Run TiDE Pipeline

##  Configure Deid job spec

TiDE has some embedded job specifications ([resource folder](src/main/resources)) that fit for most common use cases. 

If need to customize the configuration, create a new config yaml file, and use the file path as argument value of --deidConfigFile when run the tool.

## Deid Config YAML file

Sample configuration to switch on/off features

```yaml
    analytic: false
    googleDlpEnabled: false
    nerEnabled: true
```

Multiple deid actions can be grouped into same PHI category by using same __itemName__. Grouping is useful for deid quality analytics if analytic is enabled. 

Configure General Regex pattern matching, or find known PHI of the patient associated with the text. 

- general : for Phone/Fax, Email, URL, IP address, SSN
- general_number: general order, account number and general-accession
- surrogate_name : surrogate name using NameSurrogate
- surrogate_address : surrogate address using LocationSurrogate
- jitter_date_from_field : Date Anonymizer with jitter value provided in an input field
- jitter_birth_date : Date Anonymizer
- jitter_date_randomly: randomly generate jitter using hash function
- remove_age : Age Anonymizer
- remove_mrn : Mrn Anonymizer
- replace_minimumlengthword_with : find words with minimum word length
- replace_with : find word longer than 2 characters, and not in common vocabulary
- replace_strictly_with : applied strictly regardless word length, and if in common vocabulary


Configuration Example:


```yaml

name: note_deid_20180831
deidJobs:
  - jobName: stanford_deid_v1_strict
    version: v1
    spec:
      - itemName: phi_date
        action: jitter
        actionParam: 10
        fields: birth_date

      - itemName: other_date
        action: keep
        fields: '*'

      - itemName: patient_name
        action: surrogate_name
        actionParamMap: {"format":"L,F","f_zip":"zip","f_gender":"","f_dob":"birth_date"}
        fields:  pat_name, PROXY_NAME

      - itemName: patient_name
        action: surrogate_name
        actionParamMap: {"format":"F","f_zip":"zip","f_gender":"","f_dob":"birth_date"}
        fields:  PAT_FIRST_NAME, preferred_name

      - itemName: patient_mrn
        action: tag
        actionParam: mrn
        fields:  PAT_MRN_ID
        .....

```


## Set up runtime environment

You need to configure Google Cloud credential if run TiDE on Dataflow.
https://cloud.google.com/docs/authentication/getting-started


```
export GOOGLE_APPLICATION_CREDENTIALS=<gcp service account credential json file>

```


## Run Deid jobs

TiDE was created using Apache Beam programming model. So you can run TiDE on many technolgies using appropriate runners on https://beam.apache.org/documentation/runners/capability-matrix/

## Pipeline Runtime Configurations

Three types of parameters are needed for running TiDE:

1. for TiDE itself, for example, specify deid configuration
2. for Apache Beam runners
3. for running TiDE on Google Cloud Dataflow

### Pipeline runtime parameters for TiDE

|parameter|description| sample value |
|--|--|--|
|textInputFields |field name in input that contains free text | note_text| 
|textIdFields |field name in input that contains row id(s) | note_id,note_csn_id| 
| runner | type of the Apache Beam runner | DirectRunner, DataflowRunner |
|inputType    | type of the input souce. Currently supports Google Cloud Storage, Google BigQuery and local files  | gcp_gcs , gcp_bq , local |
|inputResource | Path of the file to read from | gs://mybucket/path/to/json/files/*.json|
|outputResource | Path of the output files | |
|DeidConfigFile | Name of the Deid configuration. Can use the provided configurations or external config file | deid_config_clarity.yaml |
|dlpProject   | GCP project id, if use GCP DLP service | | 
|googleDlpEnabled | Turn on/off Google DLP | true or false|


### Pipeline runtime parameters for Apache Beam runners
https://beam.apache.org/documentation/runners/dataflow/

### Pipeline runtime parameters for  Google Cloud Dataflow
https://cloud.google.com/dataflow/docs/guides/setting-pipeline-options



## Run TiDE as regular Java application
Deid jobs can be run either with Maven or Java jar.

Here is an example for reading from Google Cloud Storage and storing result to Google Cloud Storage:

```
mvn -Pdataflow-runner compile exec:java -Dexec.mainClass=com.github.susom.starr.deid.Main \
-Dexec.args="--project=<Google Project ID> \
--dlpProject=<Google Project ID for DLP API calls> \
--serviceAccount=<Service Account> \
--stagingLocation=gs://<dataflow staging bucket>/staging \
--gcpTempLocation=gs://<dataflow staging bucket>/temp \
--tempLocation=gs://<dataflow staging bucket>/temp \
--region=us-west1 --workerMachineType=n1-standard-8 --maxNumWorkers=20 --diskSizeGb=100 \
--runner=DataflowRunner \
--deidConfigFile=deid_config_clarity.yaml --inputType=gcp_gcs   \
--textIdFields="note_id,note_csn_id" \
--textInputFields="fullnote" \
--inputResource=gs://<input data bucket>/input/HNO_NOTE_TEXT_PHI_MERGED/note-input-*.json \
--outputResource=gs://<input data bucket>/NOTE_DEID_result"
```



An example of reading from local file and produce result at local

```
java -jar deid-3.0.9-SNAPSHOT.jar \
--deidConfigFile=deid_config_clarity.yaml \
--textIdFields="note_id" \
--textInputFields="note_text" \
--inputResource=/Users/wenchengli/dev/NOTE_FULL_PHI_PROV_test1_1000row.json
--outputResource=local_test2_result \

```

## Use Google DLP 

DLP can be integrated with two ways. One way is directly enable DLP in TiDE deid transform, which will call Google DLP API individually for each text row. The second way is to use Google DLP Native job to find PHIs independently from TiDe and merge findings of each parallel result into final deied-text. 

### Option one: enable DLP API Request in TiDE

Enabled Google DLP in TiDE config YAML file

```commandline
deidJobs:
  - jobName: stanford_deid_v3
    ...
    googleDlpEnabled: true
```

### Option two: Run DLP Native Job to find PHI 

``` 
java -jar deid-3.0.9-SNAPSHOT.jar \
--gcpCredentialsKeyFile=<google_credential.json> \
--projectId=<google_project_id> \
--deidConfigFile=deid_config_omop_genrep.yaml \
--inputBqTableId=<bigquery_input_text_table_id> \
--outputBqTableId=<bigquery_native_job_output_table_id> \
--idFields=note_id
--inspectFields=note_text

```
