# Credit
This code is based off the following [repo](https://github.com/sanogotech/minIO-trino-hive-docker)

# Create the project containers
```bash
podman build -t hive-poc .
```

```bash
podman-compose up minio
podman-compose up postgres
podman-compose up hive-metastore
podman-compose up trino-coordinator
```
## Load the data in minio
Access [localhost:9001](localhost:9001) with the credentials setted at `docker-compose.yml` and create a bucket "iris"      
in the bucket load the sample file in this repo "iris.parq"   

## Access trino UI
You can optionally access Trino UI at [localhost:8080](localhost:8080) to monitor the statements and executions.

## Access trino client
Run the following command to access the trino client
```bash
podman exec -it trino-coordinator trino
```

## Define the sample parquet schema
Once the connection to trino client is stablished you can run the following commands
```bash
SHOW CATALOGS;
```

We can see a catalog named `minio` this catalog was setted up at the file `etc/catalog/minio.properties`         

We can display the default schemas in minio catalog with:
```bash
SHOW SCHEMAS IN minio;
```

And create our own schema to access the iris.parq uploaded to the minio bucket
```bash
CREATE SCHEMA IF NOT EXISTS minio.iris
WITH (location = 's3a://iris/');
```

Inside our schema we can define a table which will hold the structure used in the file `iris.parq`
```bash
CREATE TABLE IF NOT EXISTS minio.iris.iris_parquet (
  sepal_length DOUBLE,
  sepal_width  DOUBLE,
  petal_length DOUBLE,
  petal_width  DOUBLE,
  class        VARCHAR
)
WITH (
  external_location = 's3a://iris/',
  format = 'PARQUET'
);
```

We can validate the table creation either on Trino UI or using the following sql statement:
```bash
SHOW TABLES IN minio.iris;
```

## Access minio data through trino and hive
Once the table is created we can access it like any other sql table
```bash
SELECT * FROM minio.iris.iris_parquet LIMIT 5;
```
We now successfully accessed the parquet file stored in minio via trino. We can perform any sql select on it.        

## Digging deeper
But why did we create the schema? Where is it stored?       
On container creation we created a postgres container let's connect to it (the password can be found in `docker-compose.yml`)

```bash
podman exec -it postgres psql -U admin -d hive_db
```
In postgres we can list the tables using
```bash
\dt
```

Some usefull queries are:
```bash
SELECT * FROM "DBS";
```

```bash
SELECT 
 t."TBL_ID", 
 t."DB_ID", 
 t."OWNER", 
 t."TBL_NAME", 
 t."TBL_TYPE",
 t."SD_ID"
FROM "TBLS" t 
 JOIN "DBS" d 
  ON t."DB_ID"= d."DB_ID" 
WHERE d."NAME" = 'iris';
```

```bash
SELECT 
 s."SD_ID",
 s."INPUT_FORMAT",
 s."LOCATION",
 s."SERDE_ID" 
FROM "TBLS" t 
 JOIN "DBS" d
  ON t."DB_ID" = d."DB_ID"
 JOIN "SDS" s 
  ON t."SD_ID" = s."SD_ID"
WHERE t."TBL_NAME" = 'iris_parquet'
 AND d."NAME"='iris';
```

```bash
SELECT c.* 
FROM "TBLS" t
 JOIN "DBS" d
  ON t."DB_ID" = d."DB_ID"
 JOIN "SDS" s
  ON t."SD_ID" = s."SD_ID"
 JOIN "COLUMNS_V2" c
  ON s."CD_ID" = c."CD_ID"
WHERE t."TBL_NAME" = 'iris_parquet'
 AND d."NAME"='iris'
ORDER by "CD_ID", "INTEGER_IDX";
```
Further research:
https://medium.com/@ahmetfurkandemir/kubernetes-with-data-engineering-approach-2-querying-to-object-storage-minio-with-trino-using-ae68ffbde9f9

