# Build an Amazon Redshift data warehouse with Etleap

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## Prerequisites

You must have an AWS account and an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) user with sufficient permissions to interact with the AWS Management Console and to create a Redshift Cluster.


## 1. Set up a Redshift Cluster

In this section we'll set up a new Redshift cluster.

### 1.1 Creating a new cluster

Log into your AWS account and [go to the Redshift Console](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters). 

- Make sure the AWS region selected is N. Virginia (us-east-1).
- Click "Create Cluster" in top right-hand corner.
- For the node type select `dc2.large`.
- For the number of nodes, select `1`.
- Under "Database configurations", enter a password that you'll remember. We will need this to connect to the cluster later on.
- Leave all other settings as they are.

Click "Create cluster".
It will take 5-10 minutes for the cluster to start up.

Once the cluster is up, you will need to make it publicly accessible.
This makes connecting to the cluster easier for the purposes of this workshop.

Select the cluster you just created, and click "Actions" in the top right corner.
Under "Manage Cluster", select "Modify publicly accessible setting" and select "Enable" in the dialog box that shows up.
Click "Save Changes".
It will take a few minutes for the cluster to become "Available".

Once the cluster is "Available", you are ready to proceed to the next step. 

### 1.2 Granting Etleap access to your new cluster

In order to allow Etleap to connect to the cluster, you'll need to allow traffic from Etleap's IP addresses.

For this, follow the instruction on our documentation page [here](https://docs.etleap.com/docs/documentation/ZG9jOjI0NTQ4MjY0-allow-inbound-access).

## 2. Connect Etleap to Redshift

We'll ETL data from two different data sources into your Redshift data warehouse. 
One source, an MongoDB source, contains JSON-formatted click event data. 
The other, a MySQL database, contains information about users. 

First, we need to setup our Redshift Cluster as a connection in Etleap.
For this setup you'll need the password you used when creating the Redshift Cluster.

We will set up the Redshift Connection using the Redshift Partner Integration:
- Go to the Redshift Console [here](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters).
- Select the cluster your just created. 
- In the top right corner, click on the "Add Partner Integration" button.
- From the list of partners, select "Etleap". It should be the first option. Click "Next".
- Leave all the settings as they are and click "Add partner".
- This will take you to the Etleap console.
  - Confirm your Redshift password. Use the same password you used when you created the Redshift Cluster. Click "Validate and Setup Connection."
  - Add your email address. This will send you a confirmation email. Click the link in the email to continue.
- Fill in your details, and click "Create Account!"
- Congratulations! Your Etleap account and Redshift connection are now ready to use.

## 3 Ingesting data from MongoDB sources

In this section, we'll configure a MongoDB source connection, and create pipelines from it.

### 3.1 Set up the MongoDB connection

Use the search box to filter for "Mongo", and click on the MongoDB icon to create a new connection.
Use the following values for the inputs:

- Name: `Website Events`
- Replica Set: `test.dev.etleap.com:27017`
- Database: `webstore`
- Authentication Database: `test` 
- Username: `devdays`
- Password: `i5eU2nZx4d`

Leave the "Use SSL" checkbox unchecked.

Click 'Save'. 

This will take you to the list of available collections in the MongoDB database.

### 3.2 Create the MongoDB Pipeline

- Select the `web_events` collection folder.
- Click 'Wrangle Data'.
- Wrangle the data. 
  - The JSON object should already be parsed by the wrangler, and each key is a column.
  - The `timestamp` column needs to be loaded as a `datetime` object in the warehouse. Click on the column header. On the right hand side, the Wrangler will suggest "Interpret `timestamp` as date and time...". Select the option, then click "Add".
  - Select the `ip` column header, and the Wrangler will suggest the "Look up geolocation..." transform. Select it, and click "Add".
- Click 'Next'.
- Select "use existing schema" and select the "public" schema. Click Next.
- Click Edit Settings, and specify the following destination values:
  - Table name: `website_events`
  - Pipeline name: `MongoDB Website Events`
- Click 'Next'.
- Click 'Start ETLing'.

The pipeline is now set up.

Click "Show Dashboard". 
You should be able to see, in the top right panel of the dashboard, that the MongoDB is now in progress.

## 4 Set up the MySQL connection and pipelines

### 4.1 Set up the MySQL connection

Click on the `+Create` button in the blue navbar.

Use the search box to filter for MySQL, and click on the MySQL icon to create a new connection. 
Use the following values as the input:

- Name: `Webstore`
- Connection Method: Direct
- Address: `test.dev.etleap.com`
- Port: `3306`
- Username: `etl`
- Password: `@1O3$zH$BYpug5LGi^5b`
- Database: `mv_webstore`
- Additional properties: Leave as their defaults.

Click 'Create Connection'

### 4.2. Set up the MySQL-to-Redshift pipeline

You'll see a list with all the tables available in the MySQL database.
We'll create pipelines for all of them in a single workflow.

- Select all the tables.
- Click 'Next'.
- Select "Use existing schema" and select the "public" schema. Click Next.
- Click "Edit Settings". Here you can change the pipelines names if you want to. Or leave them as they are and click "Next". 
- Leave all the options as their defaults in this step and click 'Next'.
- Click 'Start ETLing'.


## 5. Track ETL progress

Etleap is now ETL'ing the data from the sources to Redshift. This will take 5-10 minutes. You can monitor progress [here](https://app.etleap.com/#/activities). Once you see events saying 'Website Events loaded successfully' and 'purchases loaded successfully' you can proceed to the next section.


## 6. Run queries on Redshift

Now that we have our data in Redshift we'll run a query that uses both datasets: we'll figure out how many clicks users have on average on our site segmented by whether or not they have made at least one purchase.	

- Go to the [Redshift Console](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters) and select the cluster you created.
- Click "Query Cluster" on the top right hand of the console.
- Create a connection to the Redshift console:
  - Leave the 2 checkboxes as they selected by default
  - Cluster: leave as is
  - Database name: `dev`
  - Database user: `awsuser`

- You should now be redirected to the `Query Editor`. Enter the following query:

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

## 7. Create an Etleap Model

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

## 8. Run queries against this model

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

## 9. Delete the Redshift Cluster

- Go to the [AWS Redshift Console](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#clusters)
- Make sure the AWS region selected is N. Virginia (us-east-1).
- Select the cluster you created earlier 
- Under "Actions" in the top right corner, select "Delete cluster."
- Deselect the "Create final snapshop" option
- Click "Delete Cluster" to the remove the cluster.

This will take few minutes to delete the cluster.
