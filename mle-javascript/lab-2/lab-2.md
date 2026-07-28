# Configure AI Services and a JSON Source

## Introduction

In this lab, you will configure the AI service used by the application and create a JSON Source for the EXIF metadata stored in the database. EXIF (Exchangeable Image File Format) metadata is information embedded in an image file, such as the camera model, date, lens, exposure settings, and GPS coordinates. This information provides useful context when reviewing an image.

The application uses the configured AI service through its Static ID, `google-gemini`. Later labs call this service from JavaScript running in the database. The JSON Source exposes the metadata stored in the `EXIF_DATA` JSON column of the `SM_POSTS` table so it can be used by APEX components. It gives APEX a structured view of the JSON document and its attributes, making the metadata easier to use in reports and other components.

This lab requires Oracle AI Database 26ai and Oracle APEX 26.1.

Estimated time to complete: 10 minutes

### Objectives

In this lab, you will:

- Create an OCI Generative AI service in APEX
- Configure the service for the Google Gemini model
- Create a local JSON Source for the EXIF metadata

### Prerequisites

This lab assumes that you completed [Lab 1](../lab-1/lab-1.md) and imported the application scaffold, including its supporting objects.

You also need access to an OCI Generative AI service and permission to use the selected compartment and model. The service sends the image to the selected model for analysis. The OCI credentials are stored in the APEX workspace configuration and are not part of the application export.

## Task 1: Create the Generative AI service

1. Open **App Builder**.
2. From the App Builder home page, open **Workspace Utilities**.
3. Select **Generative AI**.
4. Click **Create** and select **OCI Generative AI Service**.

Configure the service as follows. These values tell APEX which OCI service, model, and compartment to use.

- **Name**: `google gemini` (use this exact name). 
- **Compartment ID**: The OCID of the compartment that contains the Generative AI resources. A compartment is the OCI container in which the service is authorized to access the model.
- **Region**: Select the OCI region by clicking on the name below the textbox where the service is available, for example `Germany Central (Frankfurt)`. When you select the region, APEX fills in the corresponding **Base URL**.
- **Model ID**: `google.gemini-2.5-flash`. This identifies the Gemini model that will process the image.
- - **Static ID**: `google-gemini` will be automatically filled.  The Static ID is different from the display name: it is the stable, code-friendly identifier used by the application and by the MLE module. It must remain exactly `google-gemini`.

Under **Credentials**, select an existing OCI credential configured for your workspace, or create one if your environment does not provide it. APEX uses this credential to authenticate the request without exposing secret values to the application. If you need to create the OCI API key, open the OCI Console, open your user profile, select **User Settings**, and open **API Keys**. Click **Add API Key**, download the private key, and record the user OCID, tenancy OCID, fingerprint, and private key; these values are used to create the APEX Web Credential.

In APEX, create the Web Credential from **Workspace Utilities** > **Web Credentials**, then select it in the Generative AI service configuration. Keep the private key in the credential store and never publish it.

Click **Test Connection** to verify the configuration, then click **Create** or **Apply Changes**.

The Generative AI service is now available to the application through APEX AI Services.


## Task 2: Create the JSON Source

In this task, you will connect an APEX JSON Source to the `EXIF_DATA` column in `SM_POSTS`. The source does not parse the image itself. Instead, it tells APEX where the JSON metadata is stored and which column should be used when APEX components need to read that metadata.

1. Return to **App Builder** and open the application you imported in Lab 1.

   The application must be open before you select a JSON Source because JSON Sources are application-level shared components.

2. Open **Shared Components**.
3. Under **Data Sources**, select **JSON Sources**.

   The JSON Sources page lists the JSON sources already defined for this application. In the current scaffold, no source is defined yet, so the page displays an empty list.

4. Click **Create**.

The **Create JSON Source** dialog is the first screen of the creation wizard. A JSON Source is an APEX data source that reads JSON documents from a database table or another supported location. It tells APEX where the JSON is stored and allows APEX to use the attributes inside the document without copying the data into separate columns.

Configure this screen as follows:

- **Name**: Enter `EXIF_SOURCE`. This is the descriptive name shown in APEX when you select the source later.
- **JSON Source Type**: Select **Table with JSON Columns**. This option is used because the JSON document is stored in a column of a local database table.
- **Location**: Keep **Local Database** selected. The metadata is stored in the local Oracle Database, not in a remote REST service.
- **Owner**: Select the schema that owns the `SM_POSTS` table. In the workshop environment this is `ORACLE`; in another environment, select the schema used by your APEX workspace.
- **Table with JSON Columns**: Select `SM_POSTS`. This table contains one image record per uploaded image and provides the columns shown on the next screen.

When these values are entered, click **Next**.

On the **JSON Columns** screen, APEX lists the columns from `SM_POSTS` that can contain JSON documents. APEX needs this information to identify the metadata document and inspect its attributes. The first field is required. If **JSON Column 1** is left at **- Select -** and you click **Next**, APEX displays the error **JSON Column 1 must have some value**.

- For **JSON Column 1**, select `EXIF_DATA`. This column is populated later by the MLE JavaScript module after it extracts the EXIF metadata from the uploaded image.
- Leave **JSON Schema for Column 1** and **JSON Column 2** wirth default. The application uses only one JSON column.

Click **Next** to continue.

The table `SM_POSTS` stores one image record per uploaded image. Its `EXIF_DATA` column stores the metadata as a JSON document because EXIF fields can vary between cameras and images. The JSON Source reads that document for APEX components; it does not create a second copy of the metadata.

The **Data Profile** screen is the final step of the wizard. A data profile is a description of the structure of a JSON document. It lists the attributes that APEX discovered, the data type assigned to each attribute, and the JSON path from which the value is read. For example, an attribute such as `Model` is mapped to the `Model` value inside the `EXIF_DATA` document.

Review the detected attributes and accept the defaults for this lab. The profile should show `EXIF_DATA` as the source for the EXIF attributes. No manual changes or schema file are required.

APEX generates the Static ID automatically. Click **Create** to save the JSON Source and its data profile.

APEX creates the JSON Source and its data profile for the EXIF document. The data profile describes the attributes found in the JSON, such as camera information, timestamps, and location data. APEX can now use these attributes in reports and other application components.

## Verify the configuration

Confirm that the following objects are available:

- A Generative AI service with Static ID `google-gemini`
- A JSON Source named `EXIF_SOURCE` with Static ID `exif_source`
- `SM_POSTS.EXIF_DATA` configured as the JSON column

You are now ready to continue with [Lab 3: Import third-party JavaScript modules into the database](../lab-3/lab-3.md).

## Learn More

- [APEX App Builder User's Guide: Generative AI](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/using-generative-ai.html)
- [APEX App Builder User's Guide: JSON Sources](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/managing-json-data.html)
- [Oracle Cloud Infrastructure Generative AI](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)

## Acknowledgements

- **Author** - Martin Bach, Senior Principal Product Manager
- **Contributors** - Sonja Meyer, Consulting Member of Technical Staff
- **Last Updated By/Date** - Sonja Meyer, Consulting Member of Technical Staff, July 2026
