# Adobe Experience Platform: Databricks Source Connector Tutorial
Adobe Experience Platform (AEP) supports a number of popular data warehouse and lakehouse sources out-of-the-box with native connectors. The latest addition to this collection of source connectors is one for **Azure Databricks**. The following article provides an end-to-end tutorial and deep dive information about AEP's Databricks source connector, including information on how to set up and configure an Azure Databricks instance to test it yourself.

![Developed by a Human, Not AI](/img/Developed-By-a-Human-Not-By-AI-Badge-white.svg)

## Prerequisites
* Access to an Azure-hosted [Databricks instance](https://azure.microsoft.com/en-us/products/databricks)
* Access to [Adobe Experience Platform](https://experience.adobe.com/) and [Adobe Developer Console](https://developer.adobe.com/)
* Ability to perform API calls using command line (i.e., [cURL](https://curl.se/)), custom code, or a tool like [Postman](https://www.postman.com/)
* Some sample data to load (the file used in this tutorial is available here: [Gaming_Profile.csv](/data/Gaming_Profile.csv)
---

## Synopsis and Architecture
The [Azure Databricks source connector](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/databases/databricks) for AEP allows customers to ingest data directly from Databricks into AEP without having to rely on intermediary processing and storage. AEP provides the intermediary storage behind the scenes (via the Data Landing Zone, or "DLZ") and facilitates the mapping of data across the "hops".

### High-Level Architecture
![AEP Databricks Source Architecture Diagram](/img/AEP-Databricks-Source-Diagram.svg)

### Workflow Steps
1. **The AEP dataflow service** kicks off a scheduled or ad hoc run of the Databricks source connector
    * The process checks whether the Databricks compute cluster is active. **If it isn't, a wake command is issued**
    * Once the Databricks compute cluster is active, the process continues
2. AEP dataflow service queries for the latest data from the configured source table or view in the **Databricks Unity Catalog**, optionally passing in a UTC date/time field to do incremental loads
    * Queried data is rendered to files in the **AEP DLZ**, in a special container called `dlz-databricks-container`
    * Flat file data is copied into the `/adobe-managed-staging/` system folder in the DLZ
3. Once the DLZ data has been written, the AEP dataflow service is instructed to retrieve the files, parse them, and apply a **pre-configured mapping of source fields to destination fields** in **Experience Data Model (XDM) format**
4. Mapped data is appended to a destination dataset in the **AEP Experience Lake** (aka "the data lake")
5. If the dataset in step 4 is **enabled for profile**, the new data is promoted to the real-time customer profile
6. The source files in the DLZ are purged after the flow service job completes (within seconds)

---

## Step 1 &ndash; Retrieve AEP DLZ Credentials
To kick things off, you need to retrieve the Azure Blob Storage credentials tied to the special DLZ used by AEP for the Databricks source connector. Note that this is a different set of credentials versus the traditional [Data Landing Zone source in AEP](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/cloud-storage/data-landing-zone).

There are two ways to retrieve these credentials - via the API, or UI. We'll cover both, starting with the API method.

### 1.1 &ndash; Authenticate to AEP APIs &ndash; Generate Access Token

Before calling the Experience Platform APIs, you will need to generate an access token using the credentials set up in your Adobe Developer Console API project (`client id`, `client secret`, and `scope`). For more information on setting up OAuth server-to-server authentication in Adobe Developer Console, [click here](https://developer.adobe.com/developer-console/docs/guides/authentication/ServerToServerAuthentication/implementation#setting-up-the-oauth-server-to-server-credential).

#### Authentication Request

```sh
curl -X POST 'https://ims-na1.adobelogin.com/ims/token/v3' \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode 'grant_type=client_credentials' \
    --data-urlencode 'client_id=<CLIENT_ID>' \
    --data-urlencode 'client_secret=<CLIENT_SECRET>' \
    --data-urlencode 'scope=openid,AdobeID,read_organizations,additional_info.projectedProductContext,session'
```

If everything is configured correctly, you should receive a JSON response with an **access token** value to use in subsequent API calls to AEP. 

#### Access Token Response (save the `access_token` value for later)
```json
{
    "access_token": "<ACCESS_TOKEN>",
    "token_type": "bearer",
    "expires_in": 86399
}
```

### 1.2 &ndash; Retrieve Databricks DLZ Credentials with the AEP API

Now that you have your access token, you can make the API call necessary to retrieve the Databricks DLZ credentials. 

#### DLZ Credentials Request

```sh
curl -X GET 'https://platform.adobe.io/data/foundation/connectors/landingzone/credentials?type=dlz_databricks_source' \
    --header 'Authorization: Bearer <ACCESS_TOKEN>'
    --header 'x-api-key: <CLIENT_ID>' \
    --header 'x-gw-ims-org-id: <IMS_ORG_ID>' \
    --header 'x-sandbox-name: <SANDBOX_NAME>' \
    --header 'Content-Type: application/json'
```

If everything is configured correctly, you should receive a JSON response with the DLZ credentials, including the `SASUri`, which you will use when configuring your Databricks compute cluster. The other values are useful to document if you wish to connect to the DLZ blob storage container directly.

#### DLZ Credentials Response (save the `SASUri` value for later)
```json
{
    "containerName": "dlz-databricks-container",
    "SASToken": "sv=<SV_VALUE>&si=<SI_VALUE>&sr=c&sp=racwdlm&sig=<SIG_VALUE>",
    "storageAccountName": "<STORAGE_ACCOUNT_NAME>",
    "SASUri": "https://<STORAGE_ACCOUNT_NAME>.blob.core.windows.net/dlz-databricks-container?<SAS_TOKEN>",
    "expiryDate": "<YYYY-MM-DD>"
}
```

Here is a summary of what each component of the DLZ Credentials response represents:

* **containerName** - the name of the container within your Adobe-managed Azure Storage Account that is utilized for Databricks DLZ functionality
* **SASToken** - SAS tokens are a special set of query parameters that indicate how storage resources may be accessed by a client, along with the actual signature used for authorization. You may see the following query parameters in the string:

   | Parameter | Name             | Description                                                                                                |
   |-----------|------------------|------------------------------------------------------------------------------------------------------------|
   | **sv**    |Signed Version    | Specifies the version of shared key authorization used by this SAS                                         |
   | **si**    |Signed Identifier | Unique value up to 64 characters in length that correlates to an access policy specified for the container |
   | **sr**    |Signed Resource   | Specifies which resources are accessible via the SAS (service, container, or object)                       |
   | **sp**    |Signed Permission | Specifies the signed permissions for the account SAS (read, write, delete, list, update, etc.)             |
   | **sig**   |Signature         | Signature that is used to authorize the request made with the SAS                                          |

* **storageAccountName** - the name of the specific storage account that contains your DLZ storage container. This name is globally unique within Azure
* **SASUri** - URI that contains the HTTPS endpoint for your storage account, concatenated with the SAS Token value. You will generally use this full URI when accessing the Databricks DLZ
* **expiryDate** - the specific date in which these credentials will expire

Note the `expiryDate` value in the response - these credentials are valid for 90 days by default, and will need to be refreshed prior to the expiry date to ensure continued operation of the integration. You can refresh the credentials by calling the same endpoint as above, but with an HTTP POST and an `&action=refresh` URL query parameter:

```sh
curl -X POST 'https://platform.adobe.io/data/foundation/connectors/landingzone/credentials?type=dlz_databricks_source&action=refresh' \
    --header 'Authorization: Bearer <ACCESS TOKEN>'
    --header 'x-api-key: <CLIENT_ID>' \
    --header 'x-gw-ims-org-id: <IMS_ORG_ID>' \
    --header 'x-sandbox-name: <SANDBOX_NAME>' \
    --header 'Content-Type: application/json' \
``` 

The response will look identical to the response for the GET call, just with updated credentials and a future `expiryDate` value. Note that this refresh command invalidates the previous credentials &ndash; ensure that you're ready to update your Databricks compute configuration with the new credentials.

### 1.3 &ndash; Retrieve Databricks DLZ Credentials in the AEP UI

If you're simply testing out the connector and don't need to manage DLZ credentials long term, you can also find the DLZ SAS URI in the Databricks source configuration dialog in AEP at the bottom of the screen:

![AEP Databricks Source Connector Configuration - 1 of 7](/img/aep-setup-1.png)

### 1.4 &ndash; General Notes on the Databricks DLZ in AEP

* Data Landing Zones in AEP are Azure Blob Storage containers behind the scenes, and each sandbox has its own set of containers
* You can connect to this DLZ blob storage container as you would any other Azure Blob Storage container, using PowerShell, APIs, Azure Storage Explorer, etc. - all you need are the credentials retrieved in **step 1.2** above
* The AEP Databricks source connector utilizes a folder in the DLZ called `adobe-managed-staging`
  * **This folder is not to be modified in any way, and is controlled by AEP**

---

## Step 2 &ndash; Instantiating Databricks in Azure
With DLZ credentials ready to go, you're ready to spin up and configure your Databricks instance. 

### 2.1 &ndash; Log In to the Azure Portal and Create an Azure Databricks Instance
Log in to the Azure portal, and search for **Azure Databricks**. Navigate to the Azure Databricks landing page, and click **"+ Create"** to configure your new Azure Databricks instance.

For the purposes of this tutorial, you are creating a **premium-tier Databricks workspace** in Azure. All options are default, aside from selecting the **premium-tier** over the free/standard tiers.

![Azure Databricks Instantiation - 1 of 2](/img/azure-create-1.png)

### 2.2 &ndash; Get Workspace URL and Launch into Databricks UI

Once the workspace has been provisioned, take note of the **workspace URL** and save it for later in the configuration process. Click **"Launch Workspace"** to switch to the Databricks UI.

![Azure Databricks Instantiation - 2 of 2](/img/azure-create-2.png)

---

## Step 3 &ndash; Setting Up Databricks and Loading Sample Data
After launching your new Databricks instance, you will land in the Databricks UI. The first thing you need to do is upload sample data to a volume so that it can be read into Delta tables for querying and consumption by AEP.

### 3.1 &ndash; Create Managed Volume

1. Select **"Data Ingestion"** in the left navigation menu under the **"Data Engineering"** header
2. Click **"Upload files to a volume"**
3. Download and browse for [`Gaming_Profile.csv`](/data/Gaming_Profile.csv) (or use whichever data file you want for testing)
4. Click **"Create volume"**, give your volume a name of `gaming_platform_data` and a volume type of `Managed volume`, and click **"Create"**
5. Click **"Upload"** to upload the CSV file

![Databricks Volume Creation - 1 of 3](/img/dbx-volume-create-1.png)

Once the upload completes, select **"Catalog"** in the left navigation menu and browse through the catalog directory to find your new `gaming_platform_data` volume. Inside, you should find [`Gaming_Profile.csv`](/data/Gaming_Profile.csv). You can click into the file to get a preview of its contents.

![Databricks Volume Creation - 2 of 3](/img/dbx-volume-create-2.png)

### 3.2 &ndash; Create Delta Tables and Process Data

With your source file uploaded to the new volume, now you can set up the Delta tables in the catalog, then process the CSV data into the tables.

1. Click **SQL Editor** under the "**SQL**" header
2. Copy and paste the query shown below into the query editor
    * Note that the default **"Serverless Starter Warehouse"** will spin up automatically to run the query
    * In a production Databricks environment, you would likely be using a beefier SQL Warehouse to run queries

```sql
USE CATALOG adobe_dbx_demo;
USE SCHEMA default;

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- TABLE DEFINITIONS
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
CREATE OR REPLACE TABLE Staging_Gaming_Profile (
  user_id STRING,
  username STRING,
  first_name STRING,
  last_name STRING,
  date_joined STRING,
  country STRING,
  state_province STRING,
  city STRING,
  postal_code STRING,
  preferred_platform STRING,
  customer_ltv STRING,
  email_opt_in STRING
)
USING DELTA
COMMENT 'Staging table to pre-load gaming platform users from CSV';

CREATE OR REPLACE TABLE Gaming_Profile (
  user_id            STRING,
  USERNAME           STRING,
  FIRST_NAME         STRING,
  LAST_NAME          STRING,
  date_joined        TIMESTAMP,
  country            STRING,
  state_province     STRING,
  city               STRING,
  postal_code        STRING,
  preferred_platform STRING,
  customer_ltv       DOUBLE,
  email_opt_in       BOOLEAN
)
USING DELTA
COMMENT 'Cleansed gaming platform user records';

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- LOAD CSV FROM STORAGE VOLUME INTO STAGING TABLE
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
COPY INTO Staging_Gaming_Profile
FROM '/Volumes/adobe_dbx_demo/default/gaming_platform_data/Gaming_Profile.csv'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true');

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- INSERT CLEANSED DATA INTO FINAL TABLE
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
INSERT INTO Gaming_Profile
SELECT user_id
      ,username
      ,first_name
      ,last_name
      ,TO_TIMESTAMP(date_joined, "yyyy-MM-dd'T'HH:mm:ss'Z'") as date_joined
      ,country
      ,state_province
      ,city
      ,postal_code
      ,preferred_platform
      ,CAST(customer_ltv AS DOUBLE) as customer_ltv
      ,CAST(email_opt_in AS BOOLEAN) as email_opt_in
FROM Staging_Gaming_Profile;

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- QUICK DATA CHECK
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
select *
from Gaming_Profile
limit 10;
```

> 💡 **Views Work Too!** In this example, we're using traditional materialized Delta tables. The AEP Azure Databricks connector will also support the use of **views**, for complex join, aggregation, field masking, or other similar scenarios. 

The SQL SELECT statement at the end of the query is there just to validate that your rows were processed into the `Gaming_Profile` table. If the process was successful, it should return rows and columns of populated data.

![Databricks Volume Creation - 3 of 3](/img/dbx-volume-create-3.png)

---

## Step 4 &ndash; Create Databricks Compute Cluster

The final step in your Databricks setup is to create a **compute cluster**, which AEP requires to ingest data using the source connector.

### 4.1 &ndash; Create and Start Databricks Compute Cluster

In the Databricks UI, follow these steps to configure and deploy your compute cluster:

1. Click **"Compute"** in the left navigation menu
2. On the next screen, click **"Create compute"** to launch the creation dialog
3. Select **"Single node"**
4. Set the **"Terminate after ___ minutes"** setting to a value you're comfortable with, to prevent an idle cluster from running up costs

![Databricks Compute Cluster Creation - 1 of 8](/img/dbx-cluster-create-1.png)

5. Scroll down to the **"Advanced options"** menu and click the triangle button to expand the section
6. In the **"Spark Config"** box, you will need to append the connection string to your Databricks DLZ that you retrieved in **step 1.2**.
    * The format for this string is: `fs.azure.sas.{CONTAINER_NAME}.{STORAGE_ACCOUNT_NAME}.blob.core.windows.net {SAS_TOKEN}`
    * **Pro-tip**: the exact string you need for this is available in the AEP Databricks source connector dialog (see **step 1.3** above)
      * This value is a full URL, so only extract the query string component after `.../dlz-databricks-container?` that starts with `sv=...`
   * Append this connection string on a new line under any existing configuration
    * If you wind up refreshing your SAS credentials (see **step 1.2** above), don't forget to update them in the compute configuration

![Databricks Compute Cluster Creation - 2 of 8](/img/dbx-cluster-create-2.png)

7. For the purpose of this tutorial, keep all other options default and click the **"Create compute"** button to continue.
    * Your cluster will attempt to start automatically - once it stops spinning and the state icon turns green, you're ready to continue
      * If your cluster fails to start, delete everything and start over - ensure that you have **"Single node"** selected as your compute option
      * For a production implementation, you would likely use **"Multi node"** along with a larger **"Worker type"**. We're keeping it small here for cost efficiency and simplicity.

![Databricks Compute Cluster Creation - 3 of 8](/img/dbx-cluster-create-3.png)

### 4.2 &ndash; Get Cluster ID and Create Access Token

Before you wrap up with the Databricks configuration, you need to create an access token and gather the Cluster ID of the compute cluster you just created, which you will use later when setting up the AEP source connector.

1. To get the Cluster ID, click the ellipsis next to the **"Start/Edit"** options in the top right of the screen, then click **"View JSON"**. Grab the `cluster_id` value and save it for later.

![Databricks Compute Cluster Creation - 4 of 8](/img/dbx-cluster-create-4.png)

2. To create the access token, click your user icon in the top right of the Databricks UI, then click **"Settings"**. 

![Databricks Compute Cluster Creation - 5 of 8](/img/dbx-cluster-create-5.png)

3. On the next screen, click **"Developer"** under the **"User"** menu in the left navigation. Look for the **"Access tokens"** header, and click the **"Manage"** button next to it.

![Databricks Compute Cluster Creation - 6 of 8](/img/dbx-cluster-create-6.png)

4. Give your token a descriptive comment, and a lifetime setting (measured in days; for this tutorial use the default 90 days). Click **"Generate"** to generate the token.

![Databricks Compute Cluster Creation - 7 of 8](/img/dbx-cluster-create-7.png)

5. A modal window will display your access token - **this access token will only be shown once, so make sure you copy it to a secure location**.

![Databricks Compute Cluster Creation - 8 of 8](/img/dbx-cluster-create-8.png)

Once you've documented the access token, the Databricks steps are complete, and you're ready to configure the AEP source connector.

---

## Step 5 &ndash; Configure Databricks Source Connector in AEP UI

Now that your Databricks environment is fully configured, you're on to the final major step - setting up the Databricks source connector in AEP.

### 5.1 &ndash; Pre-Work in AEP

Before continuing with the AEP Databricks source connector setup, a couple of things need to be in place: 
* An **XDM schema** that models the **Gaming Profile** data used in this tutorial
  * A file containing this schema is available [here](/src/gaming_profile.json) if you don't wish to set it up manually
* A **dataset** created in AEP using the **Gaming Profile** schema
  * Note that this data isn't required to be **profile-enabled**, but you can enable this setting if desired

Once these are in place, you are ready to proceed.

### 5.2 &ndash; Configure Source Connection to Databricks in AEP

> ❗ **Important note** - your Databricks compute cluster **must be active and running** during source connector setup in AEP. Once configured, however, the source connector will "wake" the Databricks compute cluster when a dataflow run is kicked off.

1. In your browser, navigate to [experience.adobe.com](https://experience.adobe.com)
2. Click the "waffle menu" in the top right and select **"Experience Platform"**
3. In the left navigation menu, click **"Sources"** and search for **"Azure Databricks"**
4. Click **"Set up"** in the Databricks source card:

![AEP Databricks Source Connector Configuration - 2 of 7](/img/aep-setup-2.png)

5. Select "New account" and fill out the required configuration fields:

   | Value                | Notes                                          | 
   |--------------------- | ---------------------------------------------- |
   | Name and Description | Self-explanatory                               |
   | Domain               | Workspace URL from **step 2.2**                |
   | clusterId            | Compute Cluster identifier from **step 4.2.1** |
   | Access token         | Access token from **step 4.2.5**               |
   | Database             | Name of your catalog from **step 3.2**        |

For this tutorial, the **"Database"** value will be `default` unless you applied a specific name.

![AEP Databricks Source Connector Configuration - 3 of 7](/img/aep-setup-3.png)

6. Click **"Connect to Source"** - if the connection is successful, you will see **"Connected"** with a green check mark. Click the **"Next"** button in top right to start configuration of the dataflow.

### 5.3 &ndash; Configure and Schedule Dataflow in AEP

7. On the next screen, select the `gaming_profile` table from Databricks. A preview of the data should appear. Click **"Next"** to continue.

![AEP Databricks Source Connector Configuration - 4 of 7](/img/aep-setup-4.png)

8. Under Dataset details, select the existing `Gaming_Profile` dataset. Optionally add a description, then click **"Next"**.

![AEP Databricks Source Connector Configuration - 5 of 7](/img/aep-setup-5.png)

9. On the mapping screen, you will be presented with an error indicating that the `personID` field is not mapped, which is the primary identity on the `Gaming_Profile` dataset. Find the `username` field in the source data column, click the target field it's currently assigned to (in this example, it was incorrectly mapped to `person.name.fullName`). This will open up the `Gaming_Profile` schema to the right in the UI. Scroll down and map the field to `personID`, then click **"Validate"** at the top of the page. The error should clear. Click **"Next"**

![AEP Databricks Source Connector Configuration - 6 of 7](/img/aep-setup-6.png)

10. On the scheduling screen, set the following:
  * **Frequency** of dataflow run and the **timing interval**
  * **Date/time** when you want the dataflow to **start running**
  * Whether or not you want the **first run to do a backfill** of all data from source
  * The field you wish to use for **incremental load**
    * In this case, we're using the `date_joined` date/time value as our "bookmark" for incremental loads
  
Once complete, click **"Next"** to continue.

![AEP Databricks Source Connector Configuration - 7 of 7](/img/aep-setup-7.png)

11. Review the final confirmation screen. If all looks good, click **"Finish"**

### 5.4 &ndash; Run Dataflow in AEP and Validate Results

We're nearing the home stretch - Databricks has been set up, the source connection has been created, the dataflow has been configured and scheduled, and you need to validate that the first run has completed successfully. At this point you can kick back and wait for data to flow in, but if you've read this far you're likely to want validation of the end-to-end process. Below are some details to help you rest assured that your Databricks data has made it into AEP.

#### Observing DLZ (Blob Storage) Activity

If you wish observe data in-flight through the AEP DLZ and on to AEP, you can connect to the DLZ via the [Azure Blob Storage APIs](https://learn.microsoft.com/en-us/rest/api/storageservices/blob-service-rest-api), [PowerShell](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-powershell), or [Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer/) (which we will use for this example).

> 💡 For more information on connecting to the DLZ via Azure Storage Explorer, refer to [this tutorial on DLZ](https://github.com/jeffhlewis/AEP_DataLandingZone?tab=readme-ov-file#connecting-azure-storage-explorer-to-data-landing-zone).

1. Open **Azure Storage Explorer** and connect to the Databricks DLZ container using the SAS URI value from **step 1.2**.
2. Once the AEP source dataflow connects to the Databricks compute cluster and starts receiving data, a new folder is in the container called `adobe-managed-staging`

![Databricks DLZ Run Monitoring - 1 of 2](/img/dlz-processing-1.png)

3. Inside this folder are a series of status files, as well as the actual data payload

![Databricks DLZ Run Monitoring - 2 of 2](/img/dlz-processing-2.png)

4. Upon completion of the dataflow run, these temporary objects are purged from the DLZ. Note that you do not need to interact with these temporary files in any way, they're being shown here for illustrative purposes.

#### Validating Data in AEP

5. As a final step, you may wish to run SQL queries against the dataset to validate the loaded data. To do this, launch AEP Query Service by clicking **"Queries"** in the left navigation under the **"Data management"** header. On the next screen, click **"Create Query"** in the top right to switch to the query editor. Below is a simple query that will return the record count in the dataset:

```sql
-- Return row count for dataset
select count(1) from gaming_profile;
```

![AEP Data Load Validation - 2 of 3](/img/aep-validate-2.png)

6. If you want to see the actual data in each row, run the query below to get a sampling of N number of rows (adjust the `limit` value for more)

```sql
-- Return top 10 rows from dataset (all columns)
select * from gaming_profile limit 10;
```

![AEP Data Load Validation - 2 of 3](/img/aep-validate-3.png)

7. Additionally, you can get summary dataflow execution data by clicking **"Sources"** in the left navigation under the "**Connections**" header, then clicking the **"Dataflows**" tab. Click the name of your dataflow, and you will see a summary of dataflow runs executed and records ingested/errored/etc.

![AEP Data Load Validation - 1 of 3](/img/aep-validate-1.png)

---

## Closing
That's it! To recap, you've performed the following steps in this tutorial:
* Provisioned Databricks on Azure
* Set up a catalog and Delta tables in Databricks and loaded them with data
* Spun up a Databricks compute cluster
* Configured the AEP Databricks source connector and ran a dataflow in AEP
* Used two different methods to validate that data was loaded correctly into AEP

From here, you're ready to try loading more voluminous and complex data and/or view logic from Databricks. I hope you found this end-to-end tutorial helpful in your journey to integrate Adobe Experience Platform and Databricks!

> ❗ **Important note** -  Don't forget to shut down your Databricks compute cluster to save costs, or delete it outright if you're not going to use it post-testing.

---

## Further Reading
* [Adobe Experience League > Azure Databricks Connector](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/databases/databricks)
* [Adobe Experience League > Data Landing Zone](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/cloud-storage/data-landing-zone)
* [GitHub > Data Landing Zone Tutorial](https://github.com/jeffhlewis/AEP_DataLandingZone)
* [Databricks > Delta Lake on Databricks](https://docs.databricks.com/en/delta/index.html)
* [Microsoft Learning > Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/)