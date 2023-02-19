# snowflake_cheatsheet


# Genereal

<h2>Table of Contents</h2>

[1.  Introduction on Snowflake](#1)<br>
[2. Architecture](#2)<br>
[3. Data Loading](#3)<br>
[4. Tables in Snowflake](#4)<br>




<h2 id='1'> Introduction on Snowflake </h2> 

- Analytic data warehouse
- OLAP Saas offering
- Runs completely on the cloud either AWS, Azure or GCP, has its own VPC
* decoupled storage and computation (scaled compute does not need scaled storage)

### Pricing

- Unit costs for credits and data storage determind by region
 - cost of storage(temp, transient, perm table & time-travel data & fail-safe)

- Just remember:
  - Enterprise = 90 days time travel + materialized view + multi-cluster warehouse
  - Business Critical = lots more security(HIPAA, SOC 1&2, PCI DSS)
  - VPS Edition = Own cloud service layer
  
### Supported Regions

- Multi-region account is not supported, each SF in single region

### Connection to SF

- web-base UI 
- command line client
- ODBC and JDBC
- native connector (python, Spark & Kafka)
- third party connector(DBT)
- others (Node.js, .Net)

Snowflake address

https://account_name_region_{provider}.snowflakecomputing.com

Account_name either:

- AWS = account_name.region
- GCP/Azure = account_name.region.gcp/azure

e.g http://pp12345.ap-southeast-2.snowflakecomputing.com

<h2 id='2'> Architecture </h2>

A hybrid of traditional shared-disk and shared-nothing database architecture

- using central data repo for persisted storage - All compute nodes have data access
- using MPP clusters to process queries - Each node stores portion of data locally

```{r,echo=FALSE, out.width='60%',fig.align='center'}


knitr::include_graphics('Pics/architecture-overview.png')

```
Data Storage 

* All data stored as an internal, optimised, compressed columnar format
* Can not access the data directly, only through Snowflake or SnowSQL

Query processing

* using 'Virtual warehouse' to process query
* Each VW is independent Massive Parallel processing compute cluster, doesn't share resource with other vwh and doesn't impact other

Cloud Services

- Get provision by Snowflake and within AWS, Azure or GCP
  - You don't have access to build modification
  - Instance is shared to other SF account

- Services that take care of:
  - Authentication
  - Infrastructure management
  - Metadata management
  - Query parsing and optimisation
  - Access control

Caches

- Snowflake Caches different data to improve query performance and assist in reducing cost
Metadata Cache (Cloud Service layer)
- improves compile time for queries against commonly used tables
Result Cache (Cloud Service layer)
- holds the query results
- if customers run the exact same query within 24 hours, result cache is used and not WH source being waste

Local Disk Cache or Warehouse Cache (Storage Layer)

- Cache the data used by the SQL query in its local SSD and memory
- This improves query performance if the same data was used
- Cache is deleted when the Warehouse is suspended

<h2 id='3'> Data Loading </h2>

File Location

- On Local
- On Cloud
  - AWS (Can load directly from s3 to SF)
  - Same as Azure and GCP
  
File Type

- Structured
 - Delimited files(CSV, TSV etc)

- Semi-structure
  - Json (SF can auto detect if Snappy compressed)
  - ORC
  - Parquet
  - XML
  
- Note: If files = uncompressed, on load to SF it is gzip

Best Practice

- File Sizing
  - Parquet: >3GB compressed could time out, split into 1 GB chunks
- Semi structured sizing
 - Variant data type has 16 MB compressed size limit per row
 - For Json or Avro, outer array structure can be removed using STRIP_OUTER_ARRAY

Planning Data Load

Dedicating Separate Warehouse

- Load of large data = affect query performance
  - Use separate warehouse
  - data files processed in parallel determined by servers in WH
  - split large data to scale linearly, use of small WH should be sufficient
  
Staging Data

- Name stage operation:
 - Staging = A place where the location/path of the data is stored to assist in processing the upload of files
 - Remember: Files uploaded to snowflake Staging area using PUT are automatically encrypted with 128-bit or 256 bit key
 
 Loading Data
 
 - COPY command: parallel execution
  - Supports loading by Internal Stage or S3 Bucket path
  - 1000 files max at a time
  - Identify files through pattern matching 
  - SF recommend removing the Data from the Stage once the load is completed to avoid reloading again
  
  - Note for Semi-structured
    - SF loads semi-structured data to VARIANT type column
    - Can load semi-structured into multiple columns but semi-structured data must be stored as field in structured data 
    - Use Flatten to explore compounded values into multiple rows
    
### Snowflake Stage

Type of Stages

- Default: each table and user are allocated an internal named stage
  - Specify Internal Stage in PUT command when uploading file to Snowflake
  - Specify the same stage in COPY INTO when loading data into a table
  
User Stage (Ref: @~)

  - Accessed by single user but need copying to multiple tables
  - Can't be altered or dropped
  - Can't set file format, need to specify in COPY command to table
  
Table Stage (Ref: @%)

  - Accessed by multiple users but need copying to multiple tables
  - Can't be altered or dropped 
  - Can't set file format, need to specify in COPY command to table
  - **No transformation** while loading
  
Internal Named Stages(Ref:@)

  - A Database object
  - Can load data into any table
  - Ownership of stage can be transferred
  
AWS -Bulk Load Loading from S3

- SF uses S3 Gateway Endpoint. If region bucket = Snowflake region, no route through public internet


### Snowpipe (Incremental Load)

- Enable u to loads data as soon as they are in stage (uses COPY command)
- Think of this as Snowflake way of streaming data into the table as long as you have created a Named Stage
- Generally loads older files first but doesn't guarantee (snowpipe appends to queue)
- has loading metadata to avoid duplicated loads

Billing and Usage

- User doesn't need to worry about managing vwh
- charges based on resource usage (consume credit only when active)
- view charges and usage
  - Web UI
  - SQL
  
Automating Snowpipe

- check out their AWS, GCP, Azure and REST API article

### Monitoring

Resource Monitor

- Helps control the cost and unexpected spike in credit usage
- Set Actions to Notify and Suspend if credit usage is above certain threshold
- Set intervals of monitoring
- Create by users with admin roles(ACCOUNTADMIN)
- If monitor suspends, any warehouse assigned to the monitor cannot be resumed until
   - next interval starts
   - monitor is dropped
   - credit quota is increased
   - warehouse is no longer assigned to the monitor
   - credit threshold is suspended
 
### Virtual Warehouse
 
Cluster of resource

- provides CPU, Mem and temp storage
- is active on usage of SELECT and DML
- can be stopped at any time and resized at any time(even while running)
- running queries are do not get affected, only new queries
- Query Caching
 - vwh maintain cache of table data, improves query performance 
 
 Multi-cluster warehouse
 
 - up to 10 server cluster
 - auto-suspend and auto-resume is of whole cluster not 1 server
 - can resize anytime
 - multi-cluster mode

<h2 id='4'> Tables in Snowflake </h2>

Micro-partition

- All of SF table are divided into micro partition
- Each micro-partitions are compressed columnar data
  - Max size of 16 mb compressed
  - Stored on logical hard-drive
  - Data only accessible via query through SF not directly 
  - They are immutable, can't be changed
- Order of data ingestion are used to partition these data 
- SF uses natural clustering to co-locate column with the same value or similar range

Cluster Key

- Snowflake recommends the use of Cluster key once your table grows really large (multi-terabyte range)
 - Especially when data does not cluster optimally
 
- Clustering keys are subnet of columns designed to co-locate data in the table in the same micro-partitions

Zero-copy Cloning

- Allows customers to clone their table
- snowflake references the original data(micro partition) hence zero copy 
  - when a change is made to the cloned table then a new micro-partition is created
- clone object do not inherit the source's granted privileges

Type of Table
 
- Temporary Tables
  - used to store data temporarily
  - non-permanent and exists only within the session, data is purged after session ends
  - not visible to other users and not recoverable
  - contribute to overall storage
  - belongs to DB and schema, can have the same name as another non-temp table within the same DB and schema
  - default 1 day time travel
  
- Transient tables

     - persists until dropped
     - have all functionality as permanent table but no fail-safe mode
     - default 1 day time travel
     - not visible to other users (Except thet have privileges) or session and do not support some standard features such as cloning
 
- Permanent Table
  - Have 7 days fail safe
  - Default 1 day time travel
  
- External Table
  - allow access to data stored in External Stage, No cloning, time travel or fail safe is possible
  
#### Snowflake doesn't enforce any constraints(PK, FK, unique) beside the NOT NULL constraint
  

Type of Views

- Non-materialized views
 - Named definition of query
 - Result are not stored
 
- Materialized views
 - result are stored
 - faster performance than normal views
 - contributes towards storage cost
 
-Secure View
  - Both non-materialized and materialized view can be defined as Secure view
  - Improves data privacy, hids view definition and details from unauthorised viewers
  - Performance is impacted as internal optimization is bypassed
  

### Time Travel and Fail Safe

Time Travel

- Allows DB to query, clone or restore historical data to tables, schema, or DB for up to 90 days
- Useful if data was deleted, dropped or updated

Fail safe

- Allows disaster recovery of historical data 
- Only accessible by snowflake

### Data Sharing

- only account admin can provision share
- User creates share of DB the grants object level access
- No limit on number of shares or accounts but 1 DB per share

### Access Control within Snowflake

Network Policy

- Allow access based on IP whitelist on restriction to IP blakclist

MFA

- powered by DUO system and enrolled on all accounts, customers need to enable it
- recommend enabling MFA on account admin
- use with WebUI, SnowSQL, ODBC, JDBC, Python Connector

Federated Auth and SSO

- User authenticate through external SAML 2.0-compliant identity provider
  - user don't have to log into snowflake directly
  
- As per basic introduction to SSO, a token is passed to the application that the user is trying to login to authenticate whihc in turns will open access when vertified

Access Control Models

Snowflake approach to access control uses the following models:

- Discretionary Access Control(DAC)- All object have an owner, owner can grant access to their objects
- Role-based Access Control(RBAC)- Privileges are assigned to roles, then roles to users
 - Roles are entities that contains granted privileges 

System Defined Roles

ACCOUNTADMIN 

- encapsulates SECURITYADMIN and SYSADMIN

SECURITYADMIN

- creates, modify, and drops user, roles, networks, monitor or grants

SYSADMIN

- has privileges to create WH and DB and its object
- Recommend assigning all custom roles to SYSADMIN so it has control over all objects created

PUBLIC

- Automatically grant to all user
- Can own secured objects
- Used where explicit access control is not required


### Data Security within Snowflake

End-to-end Encryption

- Snowflake encrypts all data by default, no additional cost
- Data is always protected as it is in an encrypted state
- Customer provides encrypted data to the external staging area, provide snowflake with the encryption master key when creating the named stage
- Customer provides unencrypted data to SF internal staging area, Snowflake will automatically encrypt the data

Key Rotation 

- Snowflake encrypt all data by default and keys are rotated regularly
- Users Hierarchy Key Model for its Encryption Key Management, comprises of 
  - Root Key
  - Account Master key
  - Table Master key
  - File key
  
Tri-Secret Secure and customer-managed keys(Business critical edition)

- combines the customer key with snowflake maintained key
   - create composite master key then use it to encrypts all data in the account
   - If customer key or snowflake key is revoked, data cannot be encrypted

### Stream

METADATA$Action

- indicate the DML operation

METADATA$ISUPDATE

- indicate whether the operation was part of an update. Note that streams record the differences between two offsets. If a row is added and then updated in a current offset, the delta change is a new row. The METADATA$isupdate row records a FALSE value

METADATA$ROW_ID

- specifies the unique and immutable ID for the row, which can be used to track changes to specifc rows over time

Types of Stream

- standard
- append-only
- insert-only




### addtional memort point

About clone, clone db or schema does not clone object of External table, internal stages, internal pipes
