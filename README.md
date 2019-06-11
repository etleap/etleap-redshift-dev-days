# Build an Amazon Redshift data warehouse with Etleap

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through click events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## Set up AWS VPC with Redshift

Log into your AWS account and [go to the page to create the stack for this workshop](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateUrl=https%3A%2F%2Fs3.amazonaws.com%2Fetleap-redshift-workshop%2Fcloudformation-templates%2Fcf-template.yaml&stackName=EtleapRedshiftDevDayStack). It sets up a new VPC with a Redshift cluster along with the security group rules necessary for accessing it from Etleap.

All you need to do is specify a root password for your Redshift cluster. This must consist of at least 8 alphanumeric characters only, and must have contain a lower-case letter, an upper-case letter, and a number.

Click 'Create stack'.


## Set up Etleap to ETL data into Redshift

We'll ETL data from two different data sources into your Redshift data warehouse. One source, an S3 bucket, contains JSON-formatted click event data. The other, a MySQL database, contains information about users. 

The first thing you'll need to do is log into Etleap using the credentials that were provided to you via email.

In the rest of this section we'll connect Etleap to the data sources and Redshift destination, and then we'll configure pipelines that will ETL data from the sources into the Redshift database.


### Set up the Redshift connection

Set up the Redshift connection [here](https://app.etleap.com/#/connections/new/REDSHIFT). You'll need the values from CloudFormation available on the [Outputs page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/outputs).

### Set up the S3 Input connection

Set up the S3 Input connection [here](https://app.etleap.com/#/connections/new/S3_INPUT). Use the following values:

- Name: `Website Click Events`
- Access ID: `AKIATIHTNYTXSUK6Y553`
- Secret Key: `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`
- Data Bucket: `etleap-redshift-workshop`
- Base Directory: `click-events`

Click 'Verify Permissions' and then 'Create Connection'.

### Set up the MySQL connection

Set up the MySQL connection [here](https://app.etleap.com/#/connections/new/MYSQL). Use the following values:

- Name: `Webstore`
- Connection Method: Direct
- Address: `dbtest.etleap.com`
- Port: `3306`
- Username: `etl`
- Password: `Gpte7q3IOtzP`
- Database: `webstore`

### Set up the S3-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Website Click Events' as the source.
- This page lists the files and folders available in S3. Click the radio button in the top-left to select the top-level directory.
- Click 'Wrangle Data'.
- Wrangle the data. At a minimum, specify the following rules:
  - Split out the event type from the JSON object: highlight the whitespace between the event data and the JSON object in the first row, and pick the first suggestion on the right.
  - Rename the event type column: double-click the header where it says 'split', enter 'event_type', and click enter.
  - Parse the JSON object: single-click the 'split1' column and pick the first suggestion on the right.
- Click 'Next'.
- Pick your Redshift cluster as the destination.
- Specify the following destination values:
  - Table name: `Website_Click_Events`
  - Pipeline name: `Website Click Events`
- Click 'Next'.
- Click 'Start ETLing'.

### Set up the MySQL-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Webstore' as the source.
- Pick 'purchases' as the table to import.
- Click 'Skip Wrangling'.
- Click 'Generate automatically'.
- Leave the 'Primary key' as 'id' and 'Update timestamp' as 'update_date', and click 'Next'.
- Pick 'Amazon Redshift' as the destination.
- Leave all the options as their defaults in this step and click 'Next'.
- Click 'Start ETLing'.



## Run queries on Redshift

Once the pipelines have run we'll run a query that uses both datasets: we'll figure out how many clicks users have on average segmented by whether or not they have made at least one purchase on our site.	

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshift/home?region=us-east-1#query:).
- In the 'Credentials' input, use the values from CloudFormation available on the [Outputs page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/outputs).
- Enter the following query:

```
WITH users_with_purchases AS (
  SELECT DISTINCT p.user_id
    FROM purchases
), clicks_per_user AS (
  SELECT user_id, COUNT(*) AS clicks
    FROM click_events
   GROUP BY user_id)
SELECT
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NOT NULL THEN 1 ELSE 0 END) AS with_purchase,
  SUM(CASE WHEN uwp.user_id IS NULL THEN cpu.clicks ELSE 0 END) /
  SUM(CASE WHEN uwp.user_id IS NULL THEN 1 ELSE 0 END) AS without_purchase
  FROM clicks_per_user cpu
  LEFT JOIN users_with_purchases uwp
    ON cpu.user_id = uwp.user_id
```

As you can see, users that have made a purchase have clicked about twice as much on average as those who haven't made a purchase.
