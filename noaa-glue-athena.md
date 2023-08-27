
# Walkthrough: Create Curated Dataset from NOAA Data in AWS S3

## Introduction 
This walkthrough explains how use the AWS Management Console to:
1. pinpoint desirable data stored in AWS S3 
2. create a curated dataset from the original data
3. save curated dataset back into S3, for usage in an analytics project   

These are common tasks if you are trying to filter and prepare data for an analytics or data science project. 

This walkthrough provides the specifics for using publicly available NOAA weather data and is meant to be used as a supplement to the AWS documentation.  While this walkthrough details the steps for working with a specific S3 bucket, this explore-discover-transform-write process can be applied to other S3 buckets.  The steps are:      
1) Explore the existing data files for better understanding
2) Discover the metadata (schema) of the files to enable further processing 
3) Transform the data to meet the data requirements
4) Write the curated/filtered dataset to S3

## Conventions
:moneybag: denotes cost savings tips
<br/>
:bookmark: denotes a learning based on the results 
 

## Data Requirements 
Create a curated dataset for use in an analytics project.  The project requires historical maximum air temperatures (daily) for a specific city in the United States (Knoxville, Tennessee).  The air temperature must be in degrees Fahrenheit. 

## Assumptions
The walkthrough assumes previous experience with AWS. If familiarity is lacking, links are provided for AWS documentation. 

### Prerequisites
1. An AWS account with [IAM permissions](https://aws.amazon.com/iam/) for S3, Glue, Athena
1. Familiarity with the AWS S3 console and the Actions that are
   * Available for a bucket such as [Calculate total size](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-folders.html)
   * Available for a an object such as [Query with S3 Select](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-select.html) and [Download as](https://docs.aws.amazon.com/AmazonS3/latest/userguide/accessing-an-object.html)
1. Familiarity with [AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html) and how to use [Glue Crawlers](https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html) 
1. Familiarity with Amazon Athena and how to [query the Glue Catalog](https://docs.aws.amazon.com/athena/latest/ug/querying-glue-catalog.html) from the [Athena Query Editor](https://docs.aws.amazon.com/athena/latest/ug/getting-started.html)
1. Familiarity with the [Parquet file format](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-format-parquet-home.html) 

## 1 - Explore
Initial data exploration to understand the amount of data and what it means.  

### How much data?
1. Navigate to the S3 bucket ```https://s3.console.aws.amazon.com/s3/buckets/noaa-ghcn-pds?region=us-east-1&tab=objects```
2. Use [Calculate total size](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-folders.html)


1. Use the select all checkbox to select all folders.  
1. On the right, click Actions | Calculate total size
   - Results: 265 GB and 1,012,128 files

:bookmark: From the results, there is too much data to download to a local laptop or desktop.  Leaving the data in place in the cloud is the best solution.    

### How are the files structured?
1. Return to bucket ```https://s3.console.aws.amazon.com/s3/buckets/noaa-ghcn-pds?region=us-east-1&tab=objects``` and inspect the structure. 
   - Results: Finding these in root: readme type files, ```csv```  folder, ```parquet``` folder

:bookmark: Good to know that there are readme files.  Good to know that there are parquet, will have better performance. 
 
### What does the data mean? 
1. Click the checkbox next to readme.txt and click Actions | Download as.
1. Open the file on local machine and search to find the ELEMENT for maximum temperature.  
   - Results: TMAX = Maximum temperature (tenths of degrees C)

:bookmark: The TMAX unit will require conversion to Fahrenheit.  

1. Click the checkbox next to ghcnd-stations.txt and click Actions | Download as.
1. Open the file on local machine and find the entry for 'KNOXVILLE AP'
   - Results: Weather station id = USW00013891

:bookmark: Good to see that the there is a weather station associated with the location required (Knoxville, TN). 

### Explore a single file 
Back in S3 navigate to ```s3://noaa-ghcn-pds/parquet/by_station/``` and use the search box to find the files for the required weather station, ```STATION=USW00013891```
1. Within the folder, find the "ELEMENT=TMAX/" folder. 
1. Select the blue checkbox next the parquet file.  
1. Select Actions | Query with S3 Select.  
   - For input settings, choose 'Apache Parquet'
   - For output settings, choose 'JSON'
   - Change query to ```SELECT * FROM s3object s LIMIT 10``` and click 'Run SQL query'
   - Results: Seeing fields such as: ID, DATE, DATA_VALUE, Q_FLAG, S_FLAG 
  
 :bookmark: Good to see that the weather station ID is segmented in its own folder, will save money with Glue Crawler.  
 :bookmark: Good to see that the DATA_VALUE is populated for TMAX

## 2 - Discover
A Glue Crawler can be used to discover the metadata/schema of the files so that the files can be queried using Athena.  

:moneybag: Because the [cost of Glue Crawlers](https://aws.amazon.com/glue/pricing/) is calculated on run time, minimize run time by pointing to the PARQUET TMAX files.    

Follow the [Glue Crawler Wizard Documentation](https://docs.aws.amazon.com/glue/latest/dg/console-crawlers.html). Some specifics for this walkthrough:  
1. Point the crawler to ```s3://noaa-ghcn-pds/parquet/by_station/STATION=USW00013891/ELEMENT=TMAX/``` which is an optimized Parquet file for the weather station for only the Knoxville, TN airport.  
1. Save to a new database.  ```gchn``` would be a good name for the database. 
2. After entering all the required fields, verify the crawler finished.  Should take approximately 60 seconds.  
1. Verify the new table is showing in the new database. 

## 3 - Transform
Based on readme file, TMAX is stored in (tenths of degrees C) and needs to be converted to Fahrenheit.  Athena can handle this transformation when writing out the final files to S3.

## 4 - Write 
Athena can write query results to S3 using a [CTAS command](https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html). 
You will need to pick your personal S3 location and use the S3 URL in _personal-s3-location_ 
```
CREATE table ghcn.knox_tmax_only
WITH (
  format='PARQUET', external_location='personal-s3-location'
) AS SELECT *, ((data_value / 10) * 1.8) + 32 as data_value_f FROM ghcn."element_tmax";
```

##### History
:clock3: Last verified 8/27/2023
