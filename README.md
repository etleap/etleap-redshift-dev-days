# Build an Amazon Redshift data warehouse with Etleap

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## Prerequisites

You must have an AWS account and an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) user with sufficient permissions to interact with the AWS Management Console and creating various resources. Your IAM permissions must also include access to create IAM roles and policies created by the AWS CloudFormation template.


## 1. Set up an AWS VPC with Redshift, Glue, and S3

In this section we'll set up a new VPC with the following resources:

- A Redshift cluster along with the security group rules necessary for accessing it from Etleap. 
- An S3 bucket to host our data lake.
- A Glue Catalog database to store data lake metadata.
- An IAM role that will be assigned to the Redshift cluster and used to access the data lake.
- An IAM user that will be used by Etleap to access the data lake.

Log into your AWS account and [go to the page to create the stack for this workshop](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateUrl=https%3A%2F%2Fs3.amazonaws.com%2Fetleap-redshift-workshop%2Fcloudformation-templates%2Fcf-template.yaml&stackName=EtleapRedshiftDevDayStack). 

- Make sure the AWS region selected is N. Virginia (us-east-1).
- Specify a root password for your Redshift cluster. This must consist of at least 8 alphanumeric characters only, and must contain a lower-case letter, an upper-case letter, and a number.
- The other fields have sensible defaults, but feel free to modify as you see fit.

After entering the all the parameter values: 
- Click 'Next'.
- On the next screen, enter any required tags, an IAM role, or any advanced options, and then click 'Next'.
- Review the details on the final screen, Check the box for 'I acknowledge that AWS CloudFormation might create IAM resources' and then choose 'Create' to start building the resources.
- Click 'Create stack' and wait for the creation to complete.


## 2. Connect Etleap to Redshift and the data sources

We'll ETL data from two different data sources into your Redshift data warehouse. One source, an S3 bucket, contains JSON-formatted click event data. The other, a MySQL database, contains information about users. 

The first thing you'll need to do is log into Etleap using the credentials that were provided to you via email. The email has the subject line 'Your Etleap Account'.

In the rest of this section we'll connect Etleap to the data sources and Redshift destination, so that we can create pipelines in the next section.


### 2.1. Set up the S3 Input connection

Set up the S3 Input connection [here](https://app.etleap.com/#/connections/new/S3_INPUT). Use the following values:

- Name: `Website Events`
- Access ID: (see email with subject 'Etleap and AWS Redshift Dev Days Instructions')
- Secret Key: (see email with subject 'Etleap and AWS Redshift Dev Days Instructions')
- Data Bucket: `etleap-redshift-devdays`
- Base Directory: `events`
- Additional Properties: Leave as their defaults.

Click 'Create Connection'.

### 2.2. Set up the MySQL connection

Set up the MySQL connection [here](https://app.etleap.com/#/connections/new/MYSQL). Use the following values:

- Name: `Webstore`
- Connection Method: Direct
- Address: `dbtest.etleap.com`
- Port: `3306`
- Username: `etl`
- Password: `Gpte7q3IOtzP`
- Database: `webstore`
- Additional properties: Leave as their defaults.

Click 'Create Connection'

### 2.3. Set up the Redshift connection

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1). 

Set up the Redshift connection [here](https://app.etleap.com/#/connections/new/REDSHIFT). 
- Leave the name as `Amazon Redshift`
- Connection Method: Direct
- Connection Information:
  - Address: Use the 'Value' of 'RedshiftClusterHostnameOutput' from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
  - Port: `5439`
- Authentication:
  - Username: `root`
  - Password: Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack.
- Database properties:
  - Database: `warehouse`
  - Schema: `public`
- Additional properties: Leave as their defaults.

Click 'Create Connection'.

## 3. Create Etleap ETL pipelines from the sources to Redshift

In this section we'll configure pipelines that will ETL data from the sources into the Redshift database.

### 3.1. Set up the S3-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Website Events' as the source.
- This page lists the files and folders available in S3. Click the radio button in the top-left to select the top-level directory.
- Click 'Wrangle Data'.
- Wrangle the data. At a minimum, specify the following rules:
  - Split out the event type from the JSON object: highlight the whitespace between the event data and the JSON object in the first row, and pick the suggestion on the right that says `Split data once on ' '`.
  - Rename the event type column: double-click the header where it says 'split', enter 'event_type', and click enter.
  - Parse the JSON object: single-click the 'split1' column and pick the first suggestion on the right.
- Click 'Next'.
- Pick 'Amazon Redshift' as the destination.
- Specify the following destination values:
  - Table name: `Website_Events`
  - Pipeline name: `Website Events`
- Click 'Next'.
- Click 'Start ETLing'.

### 3.2. Set up the MySQL-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Webstore' as the source.
- Pick 'purchases' as the table to import.
- Click 'Skip Wrangling'.
- Click 'Generate automatically'.
- Leave the 'Primary key' as 'id' and 'Update timestamp' as 'update_date', and click 'Next'.
- Pick 'Amazon Redshift' as the destination.
- Leave all the options as their defaults in this step and click 'Next'.
- Click 'Start ETLing'.


### 4. Track ETL progress

Etleap is now ETL'ing the data from the sources to Redshift. This will take 5-10 minutes. You can monitor progress [here](https://app.etleap.com/#/activities). Once you see events saying 'Website Events loaded successfully' and 'purchases loaded successfully' you can proceed to the next section.


## 5. Run queries on Redshift

Now that we have our data in Redshift we'll run a query that uses both datasets: we'll figure out how many clicks users have on average on our site segmented by whether or not they have made at least one purchase.	

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1).

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshift/home?region=us-east-1#query:).
- Connect to your Redshift cluster in the 'Credentials' input:
  - Cluster: Pick the cluster that begins with 'etleap-redshift-workshop'.
  - Database: `warehouse`
  - Database user: `root`
  - Database password: Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack.
- Enter the following query:

```
WITH users_with_purchases AS (
  SELECT DISTINCT p.user_id
    FROM purchases p
), clicks_per_user AS (
  SELECT split1_userid, COUNT(*) AS clicks
    FROM Website_Events
   WHERE event_type = 'Click'
   GROUP BY split1_userid)
SELECT
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN 1 ELSE 0 END) AS with_purchase,
  SUM(CASE WHEN uwp.user_id IS NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NULL THEN 1 ELSE 0 END) AS without_purchase
  FROM clicks_per_user cpu
  LEFT JOIN users_with_purchases uwp
    ON cpu.split1_userid = uwp.user_id
```
- Click 'Run query'.

As you can see, users that have made a purchase have clicked about 36% more on average as those who haven't made a purchase.


## 6. Create Etleap ETL pipelines from the sources to S3 and Glue Catalog

In this section we'll configure pipelines that will ETL data from the sources into your S3/Glue Catalog data lake.

## 6.1. Set up the S3 Data Lake connection 

Set up the S3 Data Lake connection [here](https://app.etleap.com/#/connections/new/S3_DATA_LAKE). Use the following values:

- Leave the name as `Amazon S3 Data Lake`
- Create an access ID and a secret key:
  - Make a note of the `DataLakeIAMUser` output from your CloudFormation stack.
  - Going to the [IAM users list](https://console.aws.amazon.com/iam/home?region=us-east-1#/users) and click this user.
  - Go to the 'Security credentials' tab.
  - Click 'Create access key'.
  - Input these values into the Etleap page.
- For the bucket, use the `S3DataLakeBucket` output from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
- Leave the base directory as '/'.
- For the Glue database, use the `GlueCatalogDBName` output from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
- For the Glue catalog region, specify 'us-east-1'.
- Click 'Create Connection'. Click 'Ignore and Continue' for the warning about not having data in the input path.

## 6.2. Set up the S3-to-S3/Glue pipeline

This is similar to the S3-to-Redshift pipeline, except this time the destination is S3/Glue.

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Website Events' as the source.
- This page lists the files and folders available in S3. Click the radio button in the top-left to select the top-level directory.
- Click 'Skip Wrangling'.
- Select the script from the 'Website Events' pipeline and click 'Next'.
- Select `Amazon S3 Data Lake` as the destination.
- Specify the following destination values:
  - Table name: `Website_Events`
  - Pipeline name: `Website Events - Lake`
- Click 'More destination options' and select 'Parquet' as the output format.
- Click 'Start ETLing'.

## 6.3. Connect Redshift to Glue Catalog

Now for the fun part - we're going to use Redshift Spectrum to query data stored both in Redshift and S3 in the same query. First we need to hook up Redshift to Glue Catalog:

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshift/home?region=us-east-1#query:).
- Replace the `DataLakeIAMUser` and `RedshiftSpectrumIAMRole` output from your CloudFormation stack in the query below and execute it.

```
CREATE external SCHEMA spectrumdb
FROM data catalog DATABASE '<GlueCatalogDBName>'
IAM_ROLE '<RedshiftSpectrumIAMRole>'
CREATE external DATABASE if not exists;
```

- Now we're ready to execute the query. Let's use the same query as before, but now using the data in S3 instead. The only difference in this query is the 'spectrumdb.' prefix in the from-clause in the clicks_per_user common table expression.

```
WITH users_with_purchases AS (
  SELECT DISTINCT p.user_id
    FROM purchases p
), clicks_per_user AS (
  SELECT split1_userid, COUNT(*) AS clicks
    FROM spectrumdb.Website_Events
   WHERE event_type = 'Click'
   GROUP BY split1_userid)
SELECT
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN 1 ELSE 0 END) AS with_purchase,
  SUM(CASE WHEN uwp.user_id IS NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NULL THEN 1 ELSE 0 END) AS without_purchase
  FROM clicks_per_user cpu
  LEFT JOIN users_with_purchases uwp
    ON cpu.split1_userid = uwp.user_id
```


## 7. Delete the AWS resources

- Go to [AWS CloudFormation console](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/).
- Make sure the AWS region selected is N. Virginia (us-east-1).
- Select the CloudFormation Stack you created in this workshop.
- Click Delete.
- Click “Delete Stack”.

This will take few min to delete all the resources.
