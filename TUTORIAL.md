<!---
Note: This tutorial is meant for Google Cloud Shell, and can be opened by going to
http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/mesmacosta/dlp-to-datacatalog-tags-tutorial&tutorial=TUTORIAL.md)--->
# Cloud DLP to Data Catalog Tags Tutorial

<!-- TODO: analytics id? -->
<walkthrough-author name="mesmacosta@gmail.com" tutorialName="Cloud DLP to Data Catalog Tags Tutorial" repositoryUrl="https://github.com/mesmacosta/dlp-to-datacatalog-tags-tutorial"></walkthrough-author>

## Intro

This tutorial will walk you through the execution of the Cloud DLP to Data Catalog Tags Tutorial Tutorial.

## Environment

Let's start by setting up your environment. If you already are using Big Query, Data Catalog and Cloud DLP in our project, you may skip the following steps.

## Set Up your Project

Start by setting your project ID. Replace the placeholder to your project.
```bash
gcloud config set project MY_PROJECT_PLACEHOLDER
```

Next load it in a environment variable.
```bash
export PROJECT_ID=$(gcloud config get-value project)
```

## Enable Required APIs

```bash
gcloud services enable datacatalog.googleapis.com bigquery.googleapis.com dlp.googleapis.com --project $PROJECT_ID
```

## Set up your Service Account

Create a Service Account.
```bash
gcloud iam service-accounts create dlp-to-datacatalog-tags-sa \
--display-name  "Service Account for Cloud DLP 2 Datacatalog workload" \
--project $PROJECT_ID
```

Next create a credentials folder where the Service Account will be saved.
```bash
mkdir -p ~/credentials
```

Next create and download the Service Account Key.
```bash
gcloud iam service-accounts keys create "dlp-to-datacatalog-tags-sa.json" \
--iam-account "dlp-to-datacatalog-tags-sa@$PROJECT_ID.iam.gserviceaccount.com" \
&& mv dlp-to-datacatalog-tags-sa.json ~/credentials/dlp-to-datacatalog-tags-sa.json
```

Next add Data Catalog admin role to the Service Account.
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:dlp-to-datacatalog-tags-sa@$PROJECT_ID.iam.gserviceaccount.com" \
--quiet \
--project $PROJECT_ID \
--role "roles/datacatalog.admin"
```

Next add Big Query admin role to the Service Account.
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:dlp-to-datacatalog-tags-sa@$PROJECT_ID.iam.gserviceaccount.com" \
--quiet \
--project $PROJECT_ID \
--role "roles/bigquery.admin"
```

Next add Cloud DLP admin role to the Service Account.
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:dlp-to-datacatalog-tags-sa@$PROJECT_ID.iam.gserviceaccount.com" \
--quiet \
--project $PROJECT_ID \
--role "roles/dlp.admin"
```

Next add Storage admin role to the Service Account. 
This is needed to use cloud storage to create Big Query tables.
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:dlp-to-datacatalog-tags-sa@$PROJECT_ID.iam.gserviceaccount.com" \
--quiet \
--project $PROJECT_ID \
--role "roles/storage.admin"
```

Next set up the credentials environment variable.
```bash
export GOOGLE_APPLICATION_CREDENTIALS=~/credentials/dlp-to-datacatalog-tags-sa.json
```

## Create Big Query tables

You can use the open source script [BigQuery Fake PII Creator](https://github.com/mesmacosta/bq-fake-pii-table-creator) to create BigQuery tables with example personally identifiable information (PII).

Install using PIP:
```bash
pip3 install bq-fake-pii-table-creator --user
```

Next load it to your PATH.
```bash
export PATH=~/.local/bin:$PATH
```

Next test it out.
```bash
bq-fake-pii-table-creator --help
```

Create the first table:
```bash
bq-fake-pii-table-creator --project-id $PROJECT_ID --bq-dataset-name dlp_to_datacatalog_tutorial --num-rows 5000
```

Create a second table:
```bash
bq-fake-pii-table-creator --project-id $PROJECT_ID --bq-dataset-name dlp_to_datacatalog_tutorial --num-rows 5000
```

Create a third table with obfuscated column names:
```bash
bq-fake-pii-table-creator --project-id $PROJECT_ID --bq-dataset-name dlp_to_datacatalog_tutorial --num-rows 5000 --obfuscate-col-names true
```

## Create the inspection template

Generate your OAUTH 2.0 token with `gcloud`:
```bash
TOKEN=$(gcloud auth activate-service-account --key-file=$HOME/credentials/dlp-to-datacatalog-tags-sa.json && gcloud auth print-access-token)
```

Call API using Curl. Copy this manually.
```
curl -X POST \
  https://dlp.googleapis.com/v2/projects/$PROJECT_ID/inspectTemplates \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{
  "inspectTemplate":{
    "displayName":"DLP 2 Datacatalog inspection Template",
    "description":"Scans Sensitive Data on Data Sources",
    "inspectConfig":{
      "infoTypes":[
        {
          "name":"CREDIT_CARD_NUMBER"
        },
        {
          "name":"EMAIL_ADDRESS"
        },
        {
          "name":"FIRST_NAME"
        },
        {
          "name":"IP_ADDRESS"
        },
        {
          "name":"MAC_ADDRESS"
        },
        {
          "name":"PHONE_NUMBER"
        },
        {
          "name":"PASSPORT"
        }
      ],
      "minLikelihood":"POSSIBLE",
      "includeQuote":false
    }
  },
  "templateId": "dlp-to-datacatalog-template"
}'
```
We set up a pre-defined list of DLP Info Types in the API call, you may change it or add new types using as reference: [Built-in Info types](https://cloud.google.com/dlp/docs/infotypes-reference) or by creating custom ones.

## Get the script code

The script code is available from
[this GitHub repository](https://github.com/GoogleCloudPlatform/community/tree/master/tutorials/dlp-to-datacatalog-tags).

Clone the community github:
```bash
git clone https://github.com/GoogleCloudPlatform/community
```

Next go to DLP to Datacatalog Tags tutorial folder:
```bash
cd community/tutorials/dlp-to-datacatalog-tags
```

Verify that you are at the `pom.xml` folder:
```bash
ls -l
```

## Install the BigQuery JDBC driver

Run the following commands to download the latest Simba BigQuery driver, extract it, install it with Maven, and delete the 
temporary `lib` files:

```bash
curl -o SimbaJDBCDriverforGoogleBigQuery.zip https://storage.googleapis.com/simba-bq-release/jdbc/SimbaJDBCDriverforGoogleBigQuery42_1.2.2.1004.zip && \
unzip -qu SimbaJDBCDriverforGoogleBigQuery.zip -d lib && \
mvn install:install-file  \
-Dfile=lib/GoogleBigQueryJDBC42.jar \
-DgroupId=com.simba \
-DartifactId=simba-jdbc \
-Dversion=1.0 \
-Dpackaging=jar \
-DgeneratePom=true && \
rm -rf lib
```

## Compile the script

```bash
mvn clean package -DskipTests
```

## Execute the script

For details on the CLI args, please look at the Command line parameters table on the:
[community tutorial](https://cloud.google.com/community/tutorials/dlp-to-datacatalog-tags)

Run the script:
```bash
java -cp target/dlp-to-datacatalog-tags-0.1-jar-with-dependencies.jar com.example.dlp.DlpDataCatalogTagsTutorial \
-dbType "bigquery" \
-limitMax 1000 \
-dbName dlp_to_datacatalog_tutorial \
-projectId $PROJECT_ID \
-threadPoolSize 5 \
-inspectTemplate "projects/$PROJECT_ID/inspectTemplates/dlp-to-datacatalog-template" \
-minThreshold 100
```

## Check the results of the script

After the script finishes, you can go to [Data Catalog](https://console.cloud.google.com/datacatalog) and search for sensitive
data:

![N|Solid](https://storage.googleapis.com/gcp-community/tutorials/dlp-to-datacatalog-tags/searchUI.png)

By clicking each table, you can see which columns were marked as sensitive:

![N|Solid](https://storage.googleapis.com/gcp-community/tutorials/dlp-to-datacatalog-tags/taggedTable.png)

## Cleaning up

The easiest way to avoid incurring charges to your Google Cloud account for the resources used in this tutorial is to delete 
the project you created. Otherwise delete the Big Query Dataset you created earlier.

To delete the project, follow the steps below:

1.  In the Cloud Console, [go to the Projects page](https://console.cloud.google.com/iam-admin/projects).

1.  In the project list, select the project that you want to delete and click **Delete project**.

    ![N|Solid](https://storage.googleapis.com/gcp-community/tutorials/partial-redaction-with-dlp-and-gcf/img_delete_project.png)
    
1.  In the dialog, type the project ID, and then click **Shut down** to delete the project.

## What's next

- Learn about [Cloud Data Loss Prevention](https://cloud.google.com/dlp).
- Learn about [Data Catalog](https://cloud.google.com/data-catalog).
- Try out other Google Cloud features. Have a look at our [tutorials](https://cloud.google.com/docs/tutorials).

## Congratulations!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

You've successfully finished the Cloud DLP to Data Catalog Tags Tutorial.
