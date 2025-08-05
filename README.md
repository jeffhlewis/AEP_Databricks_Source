# Adobe Experience Platform: Databricks Source Connector Tutorial
Adobe Experience Platform (AEP) supports a number of popular data warehouse and lakehouse sources out-of-the-box with native connectors. The latest addition to this collection of source connectors is one for **Databricks on Azure**. The following article provides an end-to-end tutorial and deep dive information about Adobe Experience Platform's Databricks source connector, including information on how to set up and configure an Azure Databricks instance to test it yourself.

## Prerequisites:
* Access to an Azure-hosted [Databricks instance](https://azure.microsoft.com/en-us/products/databricks)
* Access to [Adobe Experience Platform](https://experience.adobe.com/) and [Adobe Developer Console](https://developer.adobe.com/)
* Ability to perform API calls using command line (i.e, [cURL](https://curl.se/)), custom code, or a tool like [Postman](https://www.postman.com/).

---

## Synopsis and Architecture
The [Azure Databricks source connector](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/databases/databricks) for Adobe Experience Platform (aka "AEP") allows customers to ingest data directly from Databricks into AEP without having to rely in intermediary processing and storage. AEP provides the intermediary storage behind the scenes (via the Data Landing Zone, or "DLZ") and facilitates the mapping of data across the "hops".

### High-level architecture:
![AEP Databricks Source Architecture Diagram](/img/AEP-Databricks-Source-Diagram.svg)

### Workflow Steps:
1. **The Adobe Experience Platform (AEP) data flow service** kicks off a scheduled or ad-hoc run of the Databricks source connector
    * Process checks to see if Databricks compute cluster is active. **If it isn't, a wake command is issued**
    * Once the Databricks compute cluster is active, the process continues
2. AEP data flow service queries for the latest data from the configured source table or view in the **Databricks Unity Catalog**, optionally passing in a UTC date/time field to do incremental loads
    * Queried data is rendered to files in the **AEP Data Landing Zone (DLZ)**, in a special container called `dlz-databricks-container`
    * Flat file data is copied into the `/adobe-managed-staging/` system folder in the DLZ
3. Once the DLZ data has been written, the AEP data flow service is instructed to retrieve the files, parse them, and apply a **pre-configured mapping of source fields to destination fields** in **Experience Data Model (XDM) format**
4. Mapped data is appended to a destination dataset in the **AEP Experience Lake** (aka "the data lake")
5. If the dataset in step 4 is **enabled for profile**, the new data is promoted to the real-time customer profile
6. The source files in the DLZ are purged after the flow service job completes (TODO: Check timing)

---

## Step 1 - Retrieve AEP Data Landing Zone (DLZ) Credentials
To kick things off, we need to use the  retrieve the Azure Blob Storage credentials tied to the special Data Landing Zone (DLZ) used by AEP for the Databricks source connector. Note that this is a different set of credentials versus the traditional [Data Landing Zone source in AEP](https://experienceleague.adobe.com/en/docs/experience-platform/sources/connectors/cloud-storage/data-landing-zone).

There are two ways to retrieve these credentials - via the API, or UI. We'll cover both, starting with the API method.

### 1.1 - Authenticate to AEP APIs - Generate Access Token

Before calling the Experience Platform APIs, you will need to generate an access token using the credentials set up in your Adobe Developer Console API project (`client id`, `client secret`, and `scope`). For more information on setting up OAuth server-to-server authentication in Adobe Developer Console, [click here](https://developer.adobe.com/developer-console/docs/guides/authentication/ServerToServerAuthentication/implementation#setting-up-the-oauth-server-to-server-credential).

### Authentication Request:

```sh
curl -X POST 'https://ims-na1.adobelogin.com/ims/token/v3' \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode 'grant_type=client_credentials' \
    --data-urlencode 'client_id=<CLIENT ID>' \
    --data-urlencode 'client_secret=<CLIENT SECRET>' \
    --data-urlencode 'scope=openid, AdobeID, read_organizations, additional_info.projectedProductContext, session'
```

If everything is configured correctly, you should receive a JSON response with an **access token** value to use in subsequent API calls to AEP. 

### Access Token Response (200 OK):
```json
{
    "access_token": "<ACCESS TOKEN>", // <-- Save this for later
    "token_type": "bearer",
    "expires_in": 86399
}
```

### 1.2 - Retrieve Databricks Data Landing Zone Credentials with the AEP API

Now that we have our access token, we can make the API call necessary to retrieve the Databricks DLZ credentials. 

```sh
curl -X GET 'https://platform.adobe.io/data/foundation/connectors/landingzone/credentials?type=dlz_databricks_source' \
    --header 'Authorization: BEARER <ACCESS TOKEN>'
    --header 'x-api-key: <CLIENT_ID>' \
    --header 'x-gw-ims-org-id: <IMS ORG ID>' \
    --header 'x-sandbox-name: <SANDBOX NAME>' \
    --header 'Content-Type: application/json' \
```

If everything is configured correctly, you should receive a JSON response with the DLZ credentials, including the `SASUri`, which we will use when configuring our Databricks compute cluster. The other values are useful to document if you wish to connect to the DLZ blob storage container directly.

### DLZ Credentials Response
```json
{
    "containerName": "dlz-databricks-container",
    "SASToken": "sv=<SV VALUE>&si=<SI VALUE>&sr=c&sp=racwdlm&sig=<SIG VALUE>",
    "storageAccountName": "<STORAGE ACCOUNT NAME>",
    "SASUri": "https://<STORAGE ACCOUNT NAME>.blob.core.windows.net/dlz-databricks-container?<SAS TOKEN>", // <-- Save this for later
    "expiryDate": "<YYYY-MM-DD>"
}
```

* **containerName** - the name of the container within your Adobe-maanged Azure Storage Account that is utilized for Databricks Data Landing Zone functionality
* **SASToken** - SAS tokens are a special set of query parameters that indicate how storage resources may be accessed by a client, along with the actual signature used for authorization. You may see the following query parameters in the string:
  * **sv** - (Signed Version) specifies the version of Share Key authorization used by this SAS
  * **si** - (Signed Identifier) unique value up to 64 characters in length that correlates to an access policy specified for the container
  * **sr** - (Signed Resource) specifies which resources are accessible via the SAS (service, container, or object)
  * **sp** - (Signed Permission) specifies the signed permissions for the account SAS (read, write, delete, list, update, etc.)
  * **sig** - (Signature) signature that is used to authorize the request made with the SAS
* **storageAccountName** - the name of the specific storage account that contains your Data Landing Zone storage container. This name is globally unique within Azure
* **SASUri** - URI that contains the HTTPS endpoint for your storage account, concatenated with the SAS Token value. You will generally use this full URI when accessing the Databricks Data Landing Zone
* **expiryDate** - the specific date in which these credentials will expire

Note the `expiryDate` value in the response - these credentials are valid for 90 days by default, and will need to be refreshed prior to the expiry date to ensure continued operation of the integration. You can refresh the credentials by calling the same endpoint as above, but with an HTTP POST and an `&action=refresh` URL query parameter:

```sh
curl -X POST 'https://platform.adobe.io/data/foundation/connectors/landingzone/credentials?type=dlz_databricks_source&action=refresh' \
    --header 'Authorization: BEARER <ACCESS TOKEN>'
    --header 'x-api-key: <CLIENT_ID>' \
    --header 'x-gw-ims-org-id: <IMS ORG ID>' \
    --header 'x-sandbox-name: <SANDBOX NAME>' \
    --header 'Content-Type: application/json' \
``` 

The response will look identical to the response for the GET call, just with updated credentials and a future `expiryDate` value. Note that when this refresh command is issued, it will invalidate the previous credentials, so ensure that you're ready to update your Databricks compute configuration with the new credentials.

### 1.3 - Retrieve Databricks Data Landing Zone Credentials in the AEP UI

If you're simply testing out the connector and don't need to manage DLZ credentials long term, you can also find the DLZ SAS URI in the Databricks source configuration dialog in AEP at the bottom of the screen:

![AEP Databricks Source Connector Configuration - Staging Credentials](/img/aep-setup-1.png)

### 1.4 - General Notes on the Databricks Data Landing Zone in AEP

* You can connect to this DLZ blob storage container as you would any other Azure Blob storage container, using PowerShell, APIs, Azure Storage Explorer, etc. - all you need are the credentials retrieved in **step 1.2** above
* The AEP Databricks source connector utilizes a folder in the DLZ called `adobe-managed-staging`
  * **This folder is not to be modified in any way, and is controlled by AEP**

---

## Step 2 - Instantiating Databricks in Azure
With DLZ credentials ready to go, we're ready to spin up and configure our Databricks instance. 

### 2.1 - Login to Azure Portal and Create Azure Databricks Instance
Log into the Azure portal, and search for **Azure Databricks**. Navigate to the Azure Databricks landing page, and click **"+ Create"** to configure your new Azure Databricks instance.

For the purposes of this tutorial, we are creating a **premium-tier Databricks workspace** in Azure. All options are default, aside from selecting the **premium-tier** over the free/standard tiers.

![Azure Databricks Instantiation - 1 of 2](/img/azure-create-1.png)

### 2.2 - Get Workspace URL and Launch into Databricks UI

One the workspace has been provisioned, take note of the **workspace URL** and save it for later in the configuration process. Click **"Launch Workspace"** to switch to the Databricks UI.

![Azure Databricks Instantiation - 2 of 2](/img/azure-create-2.png)

---

## Step 3 - Setting up Databricks and Loading Sample Data
After launching our new Databricks instance, you will land in the Databricks UI. The first thing we need to do is upload sample data to a volume so that it can be read into Delta tables for querying and consumption by AEP.

### 3.1 - Create Managed Volume

1. Select **"Data Ingestion"** in the left rail menu under the **"Data Engineering"** header
2. Click **"Upload files to a volume"**
3. Download and browse for [`Gaming_Profile.csv`](/data/Gaming_Profile.csv) (or use whichever data file you want for testing)
4. Click **"Create volume"**, give your volume a name of `gaming_platform_demo` and a volume type of `Managed volume`, and hit **"Create"**
5. Click **"Upload"** to upload the CSV file

![Databricks Volume Creation - 1 of 3](/img/dbx-volume-create-1.png)

Once the upload completes, select **"Catalog"** in the left rail menu and browse through the catalog directory to find our new `gaming_platform_data` volume. Inside, you should find [`Gaming_Profile.csv`](/data/Gaming_Profile.csv). You can click into the file to get a preview of its contents.

![Databricks Volume Creation - 2 of 3](/img/dbx-volume-create-2.png)

### 3.2 - Create Delta Tables and Process Data

With our source file uploaded to the new volume, now we can set up the Delta tables in the catalog, then process the CSV data into the tables.

1. Click **SQL Editor** under the "**SQL**" header
2. Copy and paste the query shown below into the query editor
    * Note that the default **"Serverless Starter Warehouse"** will spin up automatically to run the query
    * In a production Databricks environment, you would likely be using a beefier SQL Warehouse to run queries

```sql
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
COMMENT "Staging table to pre-load gaming platform users from CSV"
;

CREATE OR REPLACE TABLE Gaming_Profile (
  user_id            string,
  username           string,
  first_name         string,
  last_name          string,
  date_joined        timestamp,
  country            string,
  state_province     string,
  city               string,
  postal_code        string,
  preferred_platform string,
  customer_ltv       double,
  email_opt_in       boolean
)
USING DELTA
COMMENT 'Table to store cleansed gaming platform user records'
;

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- LOAD CSV FROM STORAGE VOLUME INTO STAGING TABLE
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
COPY INTO Staging_Gaming_Profile
FROM '/Volumes/adobe_dbx_demo/default/gaming_platform_data/Gaming_Profile.csv'
FILEFORMAT = CSV
FORMAT_OPTIONS (
  'header' = 'true'
);

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
FROM Staging_Gaming_Profile
;

-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
-- QUICK DATA CHECK
-----=====-----=====-----=====-----=====-----=====-----=====-----=====-----=====
select *
from Gaming_Profile
limit 10;
```

> 💡 **Views Work Too!** In this example, we're using traditional materialized Delta tables. The AEP Azure Databricks connector will also support the use of **views**, for complex join, aggregation, field masking, or other similar scenarios. 

The SQL SELECT statement at the end of the query is there just to validate that our rows were processed into the `Gaming_Profile` table. If the process was successful

![Databricks Volume Creation - 3 of 3](/img/dbx-volume-create-3.png)

---

## Step 4 - Create Databricks Compute Cluster

The final step in our Databricks setup is to create a **compute cluster**, which AEP requires to ingest data using the source connector.

### 4.1 - Create and Start Databricks Compute Cluster

In the Databricks UI, follow these steps to configure and deploy your compute cluster:

1. Click **"Compute**" in the left rail menu
2. On the next screen, click **"Create compute**" to launch the creation dialog
3. Select **"Single node"**
4. Set the **"Terminate after ___ minutes"** setting to a value you're comfortable with, to prevent an idle cluster from running up costs

![Databricks Compute Cluster Creation - 1 of 3](/img/dbx-cluster-create-1.png)

5. Scroll down to the **"Advanced options"** menu and click the triange button to expand the section
6. In the **"Spark Config"** box, you will need to append the connection string to your Databricks Data Landing Zone that we retrieved in **step 1.2**.
    * The format for this string is: `fs.azure.sas.{CONTAINER NAME}.{STORAGE ACCOUNT NAME}.blob.core.windows.net {SAS TOKEN}`
    * **Pro-tip**: the exact string you need for this is available in the AEP Databricks source connector dialog (see **step 1.3** above)
    * Append this connection string on a new line under any existing configuration

![Databricks Compute Cluster Creation - 2 of 3](/img/dbx-cluster-create-2.png)

7. For the purpose of this tutorial, keep all other options default and click the **"Create compute"** button to continue.
    * Your cluster will attempt to start automatically - once it stops spinning and the state icon turns green, you're ready to continue
      * If your cluster fails to start, delete everything and start over - ensure that you have **"Single node"** selected as your compute option
      * For a production implementation, you would likely use **"Multi node"** along with a larger **"Worker type"**. We're keeping it small here for cost efficiency and simplicity.

![Databricks Compute Cluster Creation - 3 of 2](/img/dbx-cluster-create-3.png)

---

## Step 5 - Configure Databricks Source Connector in AEP UI

Now that our Databricks environment is fully configured, we're on to the final major step - setting up the Databricks source connector in AEP.

### 5.1 - Blah

1. In your browser, navigate to [experience.adobe.com](https://experience.adobe.com)
2. Click the "waffle menu" in the top right and select **"Experience Platform**"
3. In the left rail menu, click **"Sources"** and search for **"Azure Databricks"**
4. Click **"Set up"** in the Databricks source card:

![AEP Databricks Source Connector Configuration - Connector Setup](/img/aep-setup-2.png)


---

## Further Reading
* [blah](url)
* [blah](url)