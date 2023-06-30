# RudderStack Profile Builder                                                                                                                  
RudderStack Profile Builder (PB) is a YAML-based tool that allows you to create customer 360 profiles by stitching data together from multiple sources right in your Snowflake data warehouse (Redshift coming soon). PB can stitch user identities and user features from multiple sources including RudderStack ETL, Fivetran or other ETL tools. The resulting customer 360 table can be used to send customer data to downstream tools such as email marketing, chat, or CRM along with many other destinations using RudderStack reverse ETL. Profile Builder is also very flexible and can also be used to create profiles for users, companies, sessions or any other entity you choose. 

# Getting Started
## Clone This Repo 
```shell script
https://github.com/rudderlabs/rudderstack-profiles-basic-example.git
```
Make sure the items listed below are in your .gitignore file for hygiene and security purposes:
```
.DS_Store
output
logs
```
## Install Profile Builder

In a terminal shell, install profile builder using python pip:
```shell script
pipx install profiles-rudderstack # isolated virtual environment
-or - 
pip3 install profiles-rudderstack
```

Validate your installation of pb:
```shell script
pb version
pb --help # view list of commands
```

Create the configuration file for connection to your warehouse:
this can be modified later at (/Users/your_user/.pb/siteconfig.yaml)
```shell script
pb init connection
```

Enter your information into the prompts (example is for Snowflake):

```shell script
Select a warehouse. (Enter s for Snowflake): s
Enter Connection Name: your_connection_name <rs-profiles-test> # you will need this to ref your connection later
Enter target:  (default:dev):  # Just press enter, leaving it to default
Enter account: your_account.your_region <abc12345.us-east-1>
Enter warehouse: your_warehouse_name
Enter dbname: your_db_name
Enter schema: your_schema_name #create a schema then ref here
Enter user: your_user_name
Enter password: your_password
Enter role: your_role
Append to /Users/<user_name>/.pb/siteconfig.yaml? [y/N]
yes
```
After installing PB and configuring your connections, you need to update inputs.yaml with names of your source tables. Navigate to the inputs.yaml file and update the ```table:``` information for both tables. Keep the table names and only change the schema and database if you want to use the sample data.

```shell script

  - name: rsIdentifies
    table: PROFILES_DEMO_DB.RS_PROFILES_7_1.SAMPLE_RS_DEMO_IDENTIFIES # change this to your fully qualifed input table name 

  - name: rsTracks
    table: PROFILES_DEMO_DB.RS_PROFILES_7_1.SAMPLE_RS_DEMO_TRACKS # change this to your fully qualifed input table name 
    occurred_at_col: timestamp
```

Notice the names are mentioned as edge_sources in profiles.yaml and define specs for creating ID stitcher/feature table. This has been done for you.

ID Stitcher Example: edge_sources and user_id sticher in profiles.yaml
```shell script
models:
  - name: user_id_stitcher
    model_type: id_stitcher
    model_spec:
      edge_sources:
        - from: inputs/rsIdentifies
        - from: inputs/rsTracks
```

Feature Table Example: in profiles.yaml
```shell script
  - name: user_profile
    model_type: feature_table_model
    model_spec:
      entity_key: user
      vars:
        - entity_var:
            name: first_seen
            select: min(timestamp::date)
            from: inputs/rsTracks
```

If you are planning to use the sample data, run the command below. It will insert two sample tables into the database and schema you defined during setup:
```shell script
pb insert
```

Use this command to validate that your project will be able to access the warehouse specified in ```pb init connections``` and create objects in that warehouse.

```shell script
pb validate access
```

You can use this command to generate the SQL that will run in your warehouse. which will also tell you if there are syntax errors in your model YAML file.

```shell script
pb compile
```

If there are no errors, use this command to create the output table in your warehouse. If using the sample data this should execute in about 60 seconds:

```shell script
pb run
```

## View User Features & Profiles in Your Warehouse  


### What user profiles were created in my warehouse?
The query below will give you the user profiles view generated by PB. This view is pointed to your most up-to-date user profile table. PB automatically maintains a history of profiles tables so you can see what a profile looked like at a particular snapshot in the past.

```sql
select * from YOUR_DB.YOUR_SCHEMA.USER_PROFILE limit 5
```

| USER_MAIN_ID | VALID_AT | FIRST_SEEN | USER_LIFESPAN | DAYS_ACTIVE  |
|-------------------------------------|-------------------------|------------|-----|----|
| rid93c0681d775e73f01830351e693a610e | 2023-06-30 18:50:11.685 | 2022-11-14 | 4   | 2  |
| ridcb1b32379f00d727ee6648777534b8e5 | 2023-06-30 18:50:11.685 | 2022-11-15 | 59  | 9  |
| rid0379ebf6a4cc85cedbf436efe9bb422d | 2023-06-30 18:50:11.685 | 2022-11-18 | 56  | 11 |
| rid1bdbc498de7458039510d81b565ef6ba | 2023-06-30 18:50:11.685 | 2022-05-13 | 0   | 1  |
| rid168ce3120988c676d8c3604c0971d632 | 2023-06-30 18:50:11.685 | 2022-11-28 | 11  | 8  |

### What User IDs were stitched to make the profile?
The query below will provide a list of profile IDs connected to the other identifiers that were stitched together to create the profiles. In the example table below you will see 3 anonymous_ids, 1 user_id, and 1 email as the IDs. Note the email record was added for illustration purposes and is not in the sample dataset.

```sql
select * from YOUR_DB.YOUR_SCHEMA.USER_ID_STITCHER limit 6
```

| USER_MAIN_ID | OTHER_ID | OTHER_ID_TYPE | VALID_AT |
|-------------------------------------|----------------------------------|--------------|-------------------------|
| rid00e6b900e23df0c9aba09928ffcd0d31 | 089511773507192a39cbf1f94e34e366 | anonymous_id | 2022-06-06 19:16:45.000 |
| rid00e6b900e23df0c9aba09928ffcd0d31 | 99c6d8b0d3afc5650d3ad9b5eaa06780 | anonymous_id | 2022-06-06 19:16:45.000 |
| rid00e6b900e23df0c9aba09928ffcd0d31 | 1ef94c5bf009d0da48ac7a227aeb43be | anonymous_id | 2022-06-06 19:16:45.000 |
| rid00e6b900e23df0c9aba09928ffcd0d31 | 1945306b10849bbe946a738f6fd9372f | user_id | 2022-06-06 19:16:45.000 |
| rid00e6b900e23df0c9aba09928ffcd0d31 | sample_user@hotmail.com | email | 2022-06-06 19:16:45.000 |


### How Many IDs were part of the user profile?
This query shows the number of IDs that were used to make each profile. Notice USER ```rid00e6b900e23df0c9aba09928ffcd0d31``` had 24 different ```anonymous_ids``` and 1 ```user_id``` that went into the profile creation.
```sql
select USER_MAIN_ID as RUDDER_USER_ID,other_id_type,count (distinct other_id) as "OTHER_ID_COUNT"
from profiles_demo_db.rs_profiles_7_1.USER_ID_STITCHER
group by USER_MAIN_ID,other_id_type
order by user_main_id asc
limit 6
```
| USER_MAIN_ID                        | OTHER_ID     | COUNT_OF_IDs |
|-------------------------------------|--------------|--------------|
| rid00e6b900e23df0c9aba09928ffcd0d31 | user_id      | 1            |
| rid00e6b900e23df0c9aba09928ffcd0d31 | anonymous_id | 24           |
| rid0379ebf6a4cc85cedbf436efe9bb422d | user_id      | 1            |
| rid0379ebf6a4cc85cedbf436efe9bb422d | anonymous_id | 30           |
| rid0386089d15c9669fec23c6835fdf2ac6 | anonymous_id | 24           |
| rid0386089d15c9669fec23c6835fdf2ac6 | user_id      | 1            |

## Learn More
Profile Builder (PB) <a href="https://rudderlabs.github.io/pywht">public docs</a>
