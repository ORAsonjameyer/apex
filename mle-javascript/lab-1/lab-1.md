# Import the application scaffold to start building

## Introduction

Before you begin building the application, you'll import a prebuilt application scaffold. The scaffold provides the application structure, supporting database objects, and configuration required for the remaining labs, allowing you to focus on implementing new functionality rather than creating the project from scratch.

In this lab, you will download the scaffold and import it into the workspace you created in the earlier step.

This lab requires Oracle AI Database 26ai and Oracle APEX 26.1. Earlier versions are not supported because the application uses features introduced in these releases.

The application scaffold contains:

- The core of the APEX application itself
- Supporting objects required for the application to work

You can use any browser compatible with APEX to complete this lab.

Estimated time to complete: 5 minutes

### Objectives

In this lab, you will:

- Log into your APEX workspace
- Import the application scaffold

### Prerequisites

This lab assumes that you created an APEX workspace and downloaded the [application scaffold](../solution/scaffold.sql) file to a temporary location on your laptop.

## Task 1: Log into your APEX workspace

In a browser supported by APEX, open the URL for your APEX workspace. You must provide:

- Workspace name
- Your username
- Your password

Note that an Always-Free Autonomous AI Database service has been used to create the screenshots for this livelab. The developer experience is identical across platforms though, it does not matter where you created your workspace as long as you have one for APEX 26.1.

![Log into your APEX workspace](./images/apex-sign-in-to-workspace.png)

You are now ready to import your application.

## Task 2: Import the application scaffold

After signing in, open App Builderon the top, then click _Import_ to begin the process of importing the application scaffold.

![Import the application scaffold](./images/apex-import-application.png)

Drag and drop the application scaffold into the file upload box, or click inside the box and select the scaffold file from your local file system. Leave all the defaults in place, then click _Next_.

![Import the application scaffold](./images/apex-prepare-app-import.png)

A short confirmation dialog is displayed next. You can leave all the defaults, and click _Next_. APEX imports the application and displays the next step. The application includes supporting objects (1 Table, 1 Index and 1 Trigger) that are installed during the import process. Click on _Install Supporting Objects_ to initiate the execution of the build script.

![Confirm the installation of supporting objects](./images/apex-install-supporting-objects.png)

After a few seconds, APEX displays a confirmation that the supporting objects have been installed successfully. Click on _Install Summary_ to confirm the installation was successful. You should see _success_ for each script name listed in the table.

You can now return to the App Builder.

After the import completes, the application appears in App Builder and is ready for use in the remaining labs. In the next lab, you'll begin extending the imported application by adding the first AI-powered features.

## Learn More

- [App Builder User's Guide: Importing Export Files](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/importing-export-files.html)
- [App Builder User's Guide: Installing Supporting Objects](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/how-to-create-a-custom-packaged-application.html#GUID-0EB94EF2-9D80-4E49-AFB7-F513F4D3D092)

## Acknowledgements

- **Author** - Martin Bach, Senior Principal Product Manager
- **Contributors** - Sonja Meyer, Consulting Member of Technical Staff
- **Last Updated By/Date** - Martin Bach, Senior Principal Product Manager, August 2026
