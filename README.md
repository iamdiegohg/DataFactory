# Azure Data Factory End-to-End Pipeline Project

Welcome to my Azure Data Factory project! In this repository, I’ve built an end-to-end solution that orchestrates several pipelines, activities, and data transformations. I connected my ADF account to GitHub so that all my work is version-controlled and available for review.

Below, I explain my overall architecture and then break down each component—from the parent (E2E) pipeline down to the individual activities, datasets, and outputs.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Overview](#architecture-overview)
- [1. The End-to-End (E2E) Pipeline](#1-the-end-to-end-e2e-pipeline)
- [2. Child Pipelines](#2-child-pipelines)
  - [Manager Pipeline](#manager-pipeline)
  - [Only Selected Files Pipeline](#only-selected-files-pipeline)
  - [Data Transformation Pipeline](#data-transformation-pipeline)
  - [(Additional) GitHub Data Pull Pipeline](#additional-github-data-pull-pipeline)
- [3. Key Activities and Concepts](#3-key-activities-and-concepts)
  - [Copy Data Activity](#copy-data-activity)
  - [Get Metadata, ForEach & If Condition Activities](#get-metadata-foreach--if-condition-activities)
  - [Parameterized Datasets](#parameterized-datasets)
  - [Data Flow (Transformation) Activity](#data-flow-transformation-activity)
  - [Set Variable & Execute Pipeline Activities](#set-variable--execute-pipeline-activities)
  - [Delete Activity](#delete-activity)
- [4. Triggers](#4-triggers)
  - [Schedule Trigger](#schedule-trigger)
  - [Tumbling Window Trigger](#tumbling-window-trigger)
  - [Storage Event Trigger](#storage-event-trigger)
- [5. Datasets and Linked Services](#5-datasets-and-linked-services)
- [6. Outputs and Data Storage](#6-outputs-and-data-storage)
- [7. How to Deploy and Test](#7-how-to-deploy-and-test)
- [Closing Remarks](#closing-remarks)

---

## Project Overview

I set out to build a complete Azure Data Factory solution—from connecting to external sources (like GitHub and Data Lake Storage Gen2) to performing data movement and transformation using a series of pipelines. The final production flow is fully automated using a combination of manual triggers, schedule triggers, and storage event triggers.

---

## Architecture Overview

My solution uses the following layers:
- **Data Sources:** Files uploaded to a source container (for example, CSV files dropped by a “manager”).
- **Staging/Working Areas:** A destination container where raw files are copied.
- **Processing Area:** A reporting container where only “selected” files (those that meet a naming convention) are further processed.
- **Transformation:** A data flow that reads reporting data, applies several transformations (selecting columns, filtering, splitting, aggregation, and deriving new columns), and then writes the output.
- **Orchestration:** A parent (E2E) pipeline that executes multiple child pipelines in sequence.

---

## 1. The End-to-End (E2E) Pipeline

The **parent pipeline** (my E2E pipeline) is the central orchestrator that ties everything together. Here’s what it does:

- **Trigger:** It’s automatically triggered by a storage event trigger (or via schedule trigger, depending on the use case). For example, when a new file is dropped in the source container, the pipeline starts.
- **Execute Pipeline Activities:**  
  - It first runs the **Manager Pipeline**, which copies the raw data from the source container to a destination folder.
  - Then it calls the **Only Selected Files Pipeline** (via an Execute Pipeline activity) that validates which files meet the “fact…” naming convention.
  - Next, it executes the **Data Transformation Pipeline** to process reporting data.
  - In some cases, I also chain an activity to pull data directly from GitHub (using an HTTP-linked service) into the data lake.
- **Post-Processing:** After successful data movement and transformation, the pipeline automatically deletes the source file (using a Delete activity) so that files are not reprocessed.

This parent pipeline encapsulates the overall data ingestion, selection, transformation, and output publication processes.

---

## 2. Child Pipelines

I broke the overall workflow into several child pipelines so that each stage is modular and easier to manage.

### Manager Pipeline

- **Purpose:**  
  The Manager Pipeline is triggered (by a storage event or manually) when a file is dropped in the source container. It copies the file from the **source container** to a **destination folder**.
  
- **Key Components:**  
  - **Copy Data Activity:** Reads a CSV file from the source and copies it to a destination (raw/staging) container.
  - **Datasets:**  
    - **Source Dataset:** A CSV dataset configured for the source container.
    - **Destination Dataset:** A CSV dataset for the raw files in the destination container.
  - **Linked Services:**  
    - Data Lake Storage Gen2 service is used to connect to the storage account.

### Only Selected Files Pipeline

- **Purpose:**  
  This pipeline further processes data from the staging area. It validates and copies only those files whose names start with “fact” (as required by the reporting team).
  
- **Key Components:**  
  - **Get Metadata Activity:**  
    - Retrieves the list of files (child items) from the destination folder.
  - **ForEach Activity:**  
    - Iterates over each file from the metadata output.
  - **If Condition Activity:**  
    - Checks if the file name begins with “fact”. Only if the condition is true, it passes the file to the Copy Data Activity.
  - **Copy Data Activity:**  
    - Copies the validated file into the reporting container.
  - **Parameterized Datasets:**  
    - Both source and destination datasets are parameterized with the file name, ensuring dynamic copying without hardcoding.

### Data Transformation Pipeline

- **Purpose:**  
  Once the reporting container has the correct “fact” files, this pipeline performs data transformation.
  
- **Key Components:**  
  - **Data Flow Activity:**  
    - Uses a visual, Spark-powered data flow to perform transformations such as:
      - **Select Transformation:** Choose specific columns.
      - **Filter Transformation:** Exclude unwanted rows (e.g., filtering out Customer ID 12).
      - **Conditional Split:** Route rows into different streams (e.g., based on payment method).
      - **Derived Column:** Replace nulls or adjust column values.
      - **Aggregation:** Group data (for example, calculating the maximum product ID per customer).
    - **Sync Activity (Sink):**  
      - Writes the transformed data into an output folder in the reporting container.
  - **Datasets:**  
    - A Data Flow source dataset points to the reporting container.
    - A sink (output) dataset holds the transformed data.
  - **Debug Settings:**  
    - I configured a data flow debug cluster (with a “time to live”) so I can preview the data during development.

### Additional GitHub Data Pull Pipeline

- **Purpose:**  
  In some cases, I needed to pull data directly from an external API (in this case, a GitHub repository) into my data lake.
  
- **Key Components:**  
  - **HTTP/REST Linked Service:**  
    - Connects to GitHub by specifying a base URL and a relative URL (to fetch a CSV file).
  - **Copy Data Activity:**  
    - Copies the CSV file directly from GitHub into the destination container.
  - **Datasets:**  
    - A source dataset for HTTP (GitHub) data.
    - A destination dataset for the data lake.

---

## 3. Key Activities and Concepts

Below is a summary of the activities I used throughout the project:

### Copy Data Activity

- **Usage:**  
  To move data from one location to another (e.g., from source container to destination or from GitHub to data lake).

### Get Metadata, ForEach & If Condition Activities

- **Get Metadata:**  
  Retrieves file listings (child items) from a folder.
- **ForEach:**  
  Iterates over the file list.
- **If Condition:**  
  Checks a condition (e.g., file name starts with “fact”) and routes the processing accordingly.

### Parameterized Datasets

- I created datasets with parameters (such as the file name) so that activities (especially copy activities) can dynamically reference files without hardcoding paths.

### Data Flow (Transformation) Activity

- **Usage:**  
  To perform a series of data transformations using a GUI interface (which runs Spark code behind the scenes).  
- **Transformations Included:**  
  - Column selection  
  - Filtering  
  - Conditional splits  
  - Derived column creation  
  - Aggregation

### Set Variable & Execute Pipeline Activities

- **Set Variable:**  
  I used this activity to capture outputs (e.g., file listings from Get Metadata) and store them for later use.
- **Execute Pipeline:**  
  This activity allows one pipeline to call another (used extensively in the parent pipeline).

### Delete Activity

- **Usage:**  
  After a file is successfully copied and processed, the Delete activity removes the original file from the source to avoid duplicate processing.

---

## 4. Triggers

I implemented several types of triggers to automate pipeline execution:

### Schedule Trigger

- **Usage:**  
  Runs pipelines at specific intervals (e.g., every 15 minutes).

### Tumbling Window Trigger

- **Note:**  
  Similar to the schedule trigger but allows historical runs. (I mentioned this option, although my primary implementation used scheduled triggers.)

### Storage Event Trigger

- **Usage:**  
  Automatically triggers a pipeline as soon as a new file is added to a specified container.  
- **Implementation:**  
  I registered my storage account with Microsoft Event Grid so that when a “blob created” event occurs, the Manager Pipeline is triggered immediately.

---

## 5. Datasets and Linked Services

- **Datasets:**  
  - **CSV Source Dataset:** Points to the container where raw files are initially uploaded.
  - **Destination Dataset (Sync CSV):** For staging the raw copied files.
  - **Reporting Dataset:** For files that meet the selection criteria (i.e., those starting with “fact”).
  - **Parameterized Datasets:** Allow dynamic passing of file names via parameters.
  
- **Linked Services:**  
  - **Azure Data Lake Storage Gen2:** Used to connect to my storage account.
  - **HTTP/REST Service for GitHub:** Connects to GitHub for pulling external data.

---

## 6. Outputs and Data Storage

- **Source Container:**  
  Where files are originally dropped by the manager.
- **Destination Container:**  
  Holds the raw files copied by the Manager Pipeline.
- **Reporting Container:**  
  Contains the “selected” files (only those that start with “fact”) and also stores the transformed outputs.
- **Data Flow Output Folder:**  
  Holds the final transformed data (with confirmation “success” files generated by the data flow sink).

---

## 7. How to Deploy and Test

### Deployment

1. **GitHub Integration:**  
   I connected my Azure Data Factory instance to GitHub. The entire project (pipelines, datasets, linked services, and triggers) is version-controlled in this repository.
2. **Publishing:**  
   I used the “Publish All” button in ADF Studio to commit changes.
3. **Resource Registration:**  
   For storage event triggers, I had to manually register the Microsoft Event Grid resource provider.

### Testing

- **Manual Debug Runs:**  
  I tested each pipeline and data flow in debug mode to ensure activities executed as expected.
- **Automated Trigger Testing:**  
  I uploaded test files to the source container to validate that the storage event trigger and scheduled triggers automatically initiated the pipelines.
- **Validation:**  
  I confirmed file movement by checking the containers and verifying that the correct files were copied, transformed, and (if necessary) deleted.

---

## Closing Remarks

