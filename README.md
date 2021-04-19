# Build an Amazon Redshift data warehouse with Etleap

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## Prerequisites

You must have an AWS account and an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) user with sufficient permissions to interact with the AWS Management Console and creating various resources. Your IAM permissions must also include access to create IAM roles and policies created by the AWS CloudFormation template.


## 1. Set up an Redshift Cluster

In this section we'll set up a new Redshift cluster

Log into your AWS account and [go to the Redshift Console](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters). 

- Make sure the AWS region selected is N. Virginia (us-east-1).
- Click "Create Cluster" in top right-hand corner.
- For the node type select `dc2.large`.
- For the number of nodes, select `1`.
- Under "Database configurations", enter a password that you'll remember. We will need this to connect to the cluster later on.
- Leave all other settings as they are.

Click "Create cluster".
It will take 5-10 minutes for the cluster to start up.

## 2. Connect Etleap to Redshift and the data sources, and set up pipelines

We'll ETL data from two different data sources into your Redshift data warehouse. One source, an SFTP source, contains JSON-formatted click event data. The other, a MySQL database, contains information about users. 

The first thing you'll need to do is log into Etleap using the credentials that were provided to you via email. The email has the subject line 'Your Etleap Account'.

In the rest of this section we'll connect Etleap to the data sources and Redshift destination, so that we can create pipelines in the next section.

### 2.1. Set up the Redshift connection via the Partner Integration Console

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1). 

We will set up the Redshift Connection using the Redshift Partner Integration:
- Go to the Redshift Console [here](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters).
- Select the cluster your just created. 
- In the top right corner, click on the 'Add Partner Integration' button.
- From the list of partners, select "Etleap". It should be the first option.
- Leave all the settings as they are and click "Add partner."
- This will take you to the Etleap console.
  - Confirm your Redshift password. Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack. Click "Validate and Setup Connection." 
  - Add your email address. This will send you a confirmation email. Click the link in the email to continue.
- Fill in your details, and click "Create Account!"
- Your account and connection are now ready to use. We'll go ahead , and set up more connection.

## 2.2 Ingesting data from SFTP sources

In this section, we'll configure a SFTP connection, and create pipelines from it.

### 2.2.1 Set up the SFTP Input connection

Use the search box to filter for SFTP, and click on the SFTP icon to create a new connection.
Use the following values for the inputs

- Name: `Website Events`
- Hostname: `test.dev.etleap.com`
- Port: `2222` 
- Username: `devdays`
- Password: `i5eU2nZx4d`

Click 'Save'. 

This will take you to the list of files available in the SFTP connection.

### 2.2.2 Create the SFTP Pipeline

- Select the `events` folder, and click the checkbox in the top-left corner. This will select all the files in the `events` folder.
- Click 'Wrangle Data'.
- Wrangle the data. 
  - The JSON object should already be parsed by the wrangler, and each key is a column.
  - The `timestamp` column needs to be loaded as a `datetime` object in the warehouse. Click on the column header. On the right hand side, the Wrangler will suggest "Interpret `timestamp` as date and time...". Select the option, then click "Add".
  - Select the `ip` column header, and the Wrangler will suggest the "Look up geolocation..." transform. Select it, and click "Add".
- Click 'Next'.
- Pick 'Amazon Redshift' as the destination.
- Specify the following destination values:
  - Table name: `Website_Events`
  - Pipeline name: `Website Events`
- Click 'Next'.
- Click 'Start ETLing'.

The pipeline is now set up.

### 2.3 Set up the MySQL connection and pipelines

### 2.3.1 Set up the MySQL connection

Use the search box to filter for MySQL, and click on the MySQL icon to create a new connection. 
Use the following values as the input

- Name: `Webstore`
- Connection Method: Direct
- Address: `test.dev.etleap.com`
- Port: `3306`
- Username: `etl`
- Password: `@1O3$zH$BYpug5LGi^5b`
- Database: `mv_webstore`
- Additional properties: Leave as their defaults.

Click 'Create Connection'

### 2.3.2. Set up the MySQL-to-Redshift pipeline

- You'll see a list with all the tables available in Redshift.
- Select all the tables.
- Click 'Next'.
- Leave the settings as they are
- Pick 'Amazon Redshift' as the destination.
- Leave all the options as their defaults in this step and click 'Next'.
- Click 'Start ETLing'.


## 4. Track ETL progress

Etleap is now ETL'ing the data from the sources to Redshift. This will take 5-10 minutes. You can monitor progress [here](https://app.etleap.com/#/activities). Once you see events saying 'Website Events loaded successfully' and 'purchases loaded successfully' you can proceed to the next section.


## 5. Run queries on Redshift

Now that we have our data in Redshift we'll run a query that uses both datasets: we'll figure out how many clicks users have on average on our site segmented by whether or not they have made at least one purchase.	

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1).

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#query-editor:).
- Connect to your Redshift cluster in the 'Credentials' input:
  - Cluster: Pick the cluster that begins with 'etleapredshiftdevdaystack'.
  - Database: `warehouse`
  - Database user: `root`
  - Database password: Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack.
- Enter the following query:

```
WITH spend_per_user AS (
  SELECT u.external_id, SUM(i.price) AS spend
  FROM public.purchase p
    INNER JOIN public.line_item li ON li.purchase_id = p.id
    INNER JOIN public.item i ON i.item_id = li.item_id
    INNER JOIN public.user u ON p.user_id = u.id
  GROUP BY u.external_id
)
SELECT s.external_id, SUM(s.spend)/COUNT(e.external_id) AS spend_by_login
  FROM spend_per_user s
  INNER JOIN public.user u ON u.external_id = s.external_id
  INNER JOIN public.website_events e ON e.external_id = u.external_id
GROUP BY s.external_id
ORDER BY spend_by_login desc
LIMIT 10;
```
- Click 'Run'.

The above query takes a while to return the expected results.
Let's see if we can improve on this with a Materialized View.

## 6. Create an Etleap Model

Now that all the data is in Redshift, let's create a model to speed up the runtime of the previous query.

 - Click the 'Create' button in the top nav-bar in Etleap.
 - Pick 'Amazon Redshift' as the source.
 - Select 'Model'
 - In the query editor enter the following query:

```
SELECT u.id AS user_id, COUNT(e.external_id) AS logins
FROM public.user u, public.Website_Events e
WHERE u.external_id = e.external_id
GROUP BY u.id;
```

 - Select 'Materialized View' and click 'Next'
 - Name the table as `logins_by_user` and click 'Next'.
 - Finally, click 'Create Model' and you created a new materialized view in Etleap!

 The model model will take a few minutes to create.
 Once it is created, you can go on to next step.

## 7. Run queries against this model

Similar to section 5, once the model update is complete, run the following query:

```
WITH spend_per_user AS (
  SELECT p.user_id, SUM(i.price) AS spend
  FROM public.purchase p 
    INNER JOIN public.line_item li ON li.purchase_id = p.id
    INNER JOIN public.item i ON i.item_id = li.item_id
  GROUP BY p.user_id
)
SELECT s.user_id, SUM(s.spend)/SUM(l.logins) AS spend_by_login 
FROM spend_per_user s
  INNER JOIN public.logins_by_user l ON l.user_id = s.user_id
GROUP BY s.user_id
ORDER BY spend_by_login desc
LIMIT 10;
```

As you can see, this time the query ran considerably faster the before.
This is the power of Materialized Views.

## 8. Delete the Redshift Cluster

- Go to the [AWS Redshift Console](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters)
- Make sure the AWS region selected is N. Virginia (us-east-1).
- Select the cluster you created earlier 
- Under "Actions" in the top right corner, select "Delete cluster."
- Deselect the "Create final snapshop" option
- Click "Delete Cluster" to the remove the cluster.

This will take few minutes to delete the cluster.
