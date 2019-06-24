# Build an Amazon Redshift data warehouse with Etleap

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## 1. Set up AWS VPC with Redshift

Log into your AWS account and [go to the page to create the stack for this workshop](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateUrl=https%3A%2F%2Fs3.amazonaws.com%2Fetleap-redshift-workshop%2Fcloudformation-templates%2Fcf-template.yaml&stackName=EtleapRedshiftDevDayStack). It sets up a new VPC with a Redshift cluster along with the security group rules necessary for accessing it from Etleap. Make sure the AWS region selected is N. Virginia (us-east-1).

All you need to do is specify a root password for your Redshift cluster. This must consist of at least 8 alphanumeric characters only, and must have contain a lower-case letter, an upper-case letter, and a number.

Click 'Create stack' and wait for the creation to complete.


## 2. Connect Etleap to Redshift and the data sources

We'll ETL data from two different data sources into your Redshift data warehouse. One source, an S3 bucket, contains JSON-formatted click event data. The other, a MySQL database, contains information about users. 

The first thing you'll need to do is log into Etleap using the credentials that were provided to you via email. The email has the subject line 'Your Etleap Account'.

In the rest of this section we'll connect Etleap to the data sources and Redshift destination, so that we can create pipelines in the next section.


### 2.1. Set up the Redshift connection

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
  - Database: `workshop`
  - Schema: `public`
- Additional properties: Leave as their defaults.

Click 'Create Connection'.

### 2.2. Set up the S3 Input connection

Set up the S3 Input connection [here](https://app.etleap.com/#/connections/new/S3_INPUT). Use the following values:

- Name: `Website Events`
- Access ID: (see email with subject 'Etleap and AWS Redshift Dev Days Instructions')
- Secret Key: (see email with subject 'Etleap and AWS Redshift Dev Days Instructions')
- Data Bucket: `etleap-redshift-devdays`
- Base Directory: `events`
- Additional Properties: Leave as their defaults.

Click 'Create Connection'.

### 2.3. Set up the MySQL connection

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
  - Database: `workshop`
  - Database user: `root`
  - Database password: Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack.
- Enter the following query:

```
WITH users_with_purchases AS (
  SELECT DISTINCT p.user_id
    FROM purchases p
), clicks_per_user AS (
  SELECT userid, COUNT(*) AS clicks
    FROM Website_Events
   WHERE event_type = 'Click'
   GROUP BY userid)
SELECT
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN 1 ELSE 0 END) AS with_purchase,
  SUM(CASE WHEN uwp.user_id IS NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NULL THEN 1 ELSE 0 END) AS without_purchase
  FROM clicks_per_user cpu
  LEFT JOIN users_with_purchases uwp
    ON cpu.userid = uwp.user_id
```
- Click 'Run query'.

As you can see, users that have made a purchase have clicked about 36% more on average as those who haven't made a purchase.
