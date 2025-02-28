# Spotify ETL Pipeline Using AWS

This project implements an **ETL (Extract, Transform, Load) pipeline** that extracts data from the **Spotify API**, transforms it using **AWS Lambda**, and loads the transformed data into **Amazon S3**. The data is then cataloged using **AWS Glue** and made available for analysis using **Amazon Athena**.

---

## Architecture Overview

The architecture follows the ETL pattern:
![image](https://github.com/user-attachments/assets/4d774803-05be-4d3d-be6e-86cf1ed95248)


### Extract
- **AWS Lambda** is triggered daily by **Amazon CloudWatch** to extract data from the **Spotify API**.
- Raw JSON data is stored in **Amazon S3** under `raw_data/to_processed/`.

### Transform
- A second **AWS Lambda** function is triggered when new raw data is uploaded to S3.
- It transforms the data into three categories:
  - **Albums** (`transformed_data/album_data/`)
  - **Artists** (`transformed_data/artist_data/`)
  - **Songs** (`transformed_data/songs_data/`)
- The transformed data is saved as CSV files in S3.

### Load
- **AWS Glue Crawler** infers the schema from the transformed data and updates the **Glue Data Catalog**.
- Data is then available for querying using **Amazon Athena**.

---

## Tech Stack and Services Used

- **Spotify API**: Data source for playlists and tracks.
- **AWS Lambda**: Serverless computing for extraction and transformation.
- **Amazon S3**: Storage for raw and transformed data.
- **AWS CloudWatch**: Scheduled trigger for data extraction.
- **AWS Glue**: Schema inference and cataloging.
- **Amazon Athena**: Data analytics and querying.
- **Python**: Programming language for Lambda functions.
- **Pandas**: Data transformation and CSV processing.


## Data Flow

### Extraction:
- CloudWatch triggers the `extract_lambda.py` Lambda function daily.
- Extracts playlist tracks from the **Spotify API**.
- Saves raw JSON data to S3: `raw_data/to_processed/`.

### Transformation:
- S3 trigger activates `transform_lambda.py` when new JSON files are uploaded.
- Transforms JSON data into structured CSV files:
  - **Albums**
  - **Artists**
  - **Songs**
- Saves the transformed data to S3:
    ```
    transformed_data/album_data/
    transformed_data/artist_data/
    transformed_data/songs_data/
    ```

### Loading and Analytics:
- **AWS Glue Crawler** infers the schema from transformed data and updates the Glue Data Catalog.
- **Amazon Athena** queries the data using SQL.

## Setup and Configuration

### Prerequisites
- **AWS Account** with the following services:
  - S3, Lambda, CloudWatch, Glue, Athena
- **Spotify Developer Account** for API credentials.

---

### 1. Create and Configure S3 Buckets
- **Bucket Name:** `spotify-etl-python-ali`
- **Folder Structure:**
    ```bash
    raw_data/to_processed/
    transformed_data/album_data/
    transformed_data/artist_data/
    transformed_data/songs_data/
    raw_data/processed/
    ```

---

### 2. Set Up Spotify API Credentials
- Go to [Spotify Developer Dashboard](https://developer.spotify.com/dashboard/applications).
- Create an app to get:
  - `client_id`
  - `client_secret`

- **Add these as Lambda Environment Variables:**
    ```
    client_id
    client_secret
    ```

---

### 3. AWS Lambda Functions

- **Extract Lambda (`extract_lambda.py`)**
  - Extracts playlist data from Spotify and saves as raw JSON in S3.
  - **Trigger:** CloudWatch Event for daily execution.

- **Transform Lambda (`transform_lambda.py`)**
  - Transforms raw JSON data into structured CSV files.
  - **Trigger:** S3 PUT Event when new JSON files are added to `raw_data/to_processed/`.

---


### 4. CloudWatch Event Rule
- Create a **CloudWatch Event Rule** to trigger the `extract_lambda.py` function daily.
- **Schedule Expression:**
    ```scss
    cron(0 0 * * ? *)
    ```
    This triggers the Lambda function at midnight UTC.

---


### 5. AWS Glue Crawler
- Create a **Glue Crawler** to infer the schema from the transformed CSV files.
- **Data source:** S3 path to the transformed data folders.
- **Target:** Glue Data Catalog.
- **Schedule** the crawler to run after the transformation is complete.

---

### 6. Amazon Athena
- Configure **Amazon Athena** to query the cataloged data.
- **Example Queries:**
    ```sql
    SELECT * FROM album_data LIMIT 10;
    SELECT artist_name, COUNT(*) as song_count FROM artist_data GROUP BY artist_name;
    ```
## IAM Roles and Permissions

### Lambda Execution Role
- **Required Policies:**
  - `AWSLambdaBasicExecutionRole`
  - `AmazonS3FullAccess`
  - `AWSGlueConsoleFullAccess`
  - `AthenaFullAccess`

## Trigger Setup

### 1. CloudWatch Trigger for Extract Lambda

- **Purpose:** 
  - Triggers the `extract_lambda.py` function daily to extract data from the **Spotify API**.

- **Service Used:** 
  - **Amazon CloudWatch Events**

- **Configuration Details:**
  - **Target:** Extract Lambda Function (`extract_lambda.py`)
  - **Schedule Expression:**
    ```scss
    cron(0 0 * * ? *)
    ```
    - This triggers the Lambda function **daily at midnight UTC**.
  - **Event Source:** CloudWatch Event Rule

- **Setup Steps:**
  1. Go to **CloudWatch Console** → **Rules**.
  2. Click **Create Rule**.
  3. **Event Source:** Select **Schedule**.
  4. Enter the **cron expression**: `cron(0 0 * * ? *)`.
  5. **Target:** Choose the Extract Lambda function.
  6. Click **Create Rule** to save.

---

### 2. S3 Trigger for Transform Lambda

- **Purpose:** 
  - Triggers the `transform_lambda.py` function whenever a new raw JSON file is uploaded to S3.

- **Service Used:** 
  - **Amazon S3 Event Notifications**

- **Configuration Details:**
  - **Bucket Name:** `spotify-etl-python-ali`
  - **Event Type:** `PUT` (Object Created)
  - **Prefix:** `raw_data/to_processed/`
  - **Suffix:** `.json`
  - **Target:** Transform Lambda Function (`transform_lambda.py`)

- **Setup Steps:**
  1. Go to **S3 Console** → Select the bucket: `spotify-etl-python-ali`.
  2. Navigate to **Properties** → **Event notifications**.
  3. Click **Create event notification**.
  4. **Name:** `RawDataUploadTrigger`
  5. **Event Type:** `All object create events`
  6. **Prefix:** `raw_data/to_processed/`
  7. **Suffix:** `.json`
  8. **Destination:** Choose **Lambda Function** and select the **Transform Lambda**.
  9. Click **Save changes**.

---

### 3. Optional: Glue Crawler Trigger for Loading Stage

- **Purpose:** 
  - Automates the schema inference and catalog update when new transformed data is uploaded.
  - Not mandatory but useful for keeping the Glue Data Catalog up-to-date.

- **Service Used:** 
  - **AWS Glue Crawler**

- **Configuration Details:**
  - **Event Source:** S3 PUT Event
  - **Target:** Glue Crawler

- **Setup Steps (Using EventBridge):**
  1. Go to **EventBridge Console** → **Rules**.
  2. Click **Create Rule**.
  3. **Event Source:** **S3**.
  4. **Event Type:** **Object Created**.
  5. **Bucket Name:** `spotify-etl-python-ali`
  6. **Prefix:** `transformed_data/`
  7. **Target:** Choose **Glue Crawler**.
  8. Select the Glue Crawler configured for this project.
  9. Click **Create Rule** to save.

---

## Summary of Triggers

1. **CloudWatch Event Rule:** 
   - **Triggers Extract Lambda** daily at midnight.
2. **S3 Event Notification:** 
   - **Triggers Transform Lambda** when new JSON files are added to `raw_data/to_processed/`.
3. **(Optional) EventBridge Rule:** 
   - **Triggers Glue Crawler** when transformed data is saved to `transformed_data/`.

This setup ensures a seamless ETL workflow from data extraction to transformation and loading into **AWS Glue Data Catalog**, ready for querying in **Amazon Athena**.

## Testing and Execution

### 1. Manual Trigger
- Manually trigger the extract function from the **Lambda Console**.

### 2. Upload Test JSON File
- Upload a sample `.json` file to `raw_data/to_processed/` in S3 to test the transform function.

### 3. Query Data in Athena
- Go to **Amazon Athena Console** and query the transformed data.

