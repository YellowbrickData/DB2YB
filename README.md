
# DB2YB Notebook

---

## Overview

**DB2YB** is a Jupyter notebook designed to run directly within the Databricks environment. It can:

For each table matching a list of Regex patterns in Databricks:

1. Read the table definitions from Databricks
2. Translate them to Yellowbrick format
3. Create the schema if necessary in Yellowbrick
4. Drop and recreate the table in Yellowbrick
5. Perform a parallel data extract from Databricks into an S3 bucket.  NOTE: Set useS3 to FALSE to use spark write
6. Load the data quickly into Yellowbrick using the `LOAD` command
7. Validate that the row counts match between source and destination
8. Remove the data from S3

Additional features:

- Limit rows copied to Yellowbrick using the limit parameter
- Intercept and edit the automated DDL conversion using write_ddl and read_ddl. 
- Add a where clause to copy only selected rows
- Use an append mode that doesn't recreate the destination tables and only copies new matching rows

> **WARNING**: This can be destructive as it creates destination tables or deletes rows from them in append mode.

---

## Prerequisites

You must use a **compute cluster** (not serverless) to run the notebook. The cluster should be:

- Dedicated
- Runtime Version: 15.4 LTS (includes Apache Spark 3.5.0, Scala 2.12)
- "Use Photon" enabled
- Worker type: Depends on source data size and performance requirements

Ensure you have connectivity properly configured between Databricks and Yellowbrick, especially for private Yellowbrick instances. See "VPC Peering" in the Yellowbrick documentation for more information.

You also need an S3 bucket set up with an API Access Key if useS3 is true.   Using S3 speeds data transfers 20x for tables over 1M rows.

---

## Running the Code Interactively

The notebook is divided into separate code blocks. This section explains each block.

### Code Block 1: Initialization Info

Edit this section to include connection info for Databricks, the S3 bucket for temporary storage, and the Yellowbrick database instance.

- **Lines 1-14**: Edit with your connection information (examples are provided in comments).
- **Lines 16-17**: Typically computed automatically from the values entered above.
- **Line 19**: Controls the number of CSV files created during the extract. A value of 20 is typically good for ~100 million rows. Adjust as needed.

| Constant | Description | Example |
|:---------|:------------|:--------|
| `YB_HOST` | Fully qualified Yellowbrick host name | `ybhost.elb.us-east-2.amazonaws.com` |
| `YB_USER` | Yellowbrick username | `JoeDBA@yellowbrickcloud.com` |
| `YB_PASSWORD` | Yellowbrick password | `SuperS3cret!` |
| `YB_DATABASE` | Yellowbrick database name | `prod_db` |
| `YB_SCHEMA` | Yellowbrick schema | `gold` |
| `DB_DATABASE` | Databricks database/catalog name | `corp` |
| `DB_SCHEMA` | Databricks schema | `silver` |
| `AWS_REGION` | AWS region name | `us-east-1` |
| `AWS_ACCESS_KEY_ID` | AWS S3 access key ID | `AVIAGI4VJJXF34LCFSN2` |
| `AWS_SECRET_KEY` | AWS S3 secret access key | `0Ps+NeWy1uf5cXfzg8qZoABdwv9oBbJh2Q0n2pB4` |
| `BUCKET_NAME` | S3 bucket name | `my_bucket` |
| `SPARK_PARTITIONS` | Number of spark paritions | Increase if using more nodes |
| `MIN_S3_ROWS` | Minimum number of rows to use S3 | Using S3 has a startup time.  For small tables, spark write is faster |


### Code Block 2: DB2YB Code

Defines the `DB2YB` function. Running this block defines the function but does not execute it.

### Code Blocks 3-7: Examples

Sample calls to `DB2YB`. You can edit and run these examples based on your specific requirements.

---

## Main Command: DB2YB Function

The notebook defines a `DB2YB` function that accepts named parameters to control operation.

### Arguments

| Argument | Description | Example |
|:---------|:------------|:--------|
| `table_patterns` | A python list of regex patterns for table matching | `table_patterns=['wid.*','big_table']` |
| `limit` | Limits the number of rows transferred to the number specified | `limit=20` |
| `where` | Adds a WHERE clause to limit rows (see Predicate Specification below) | `where='[Hour]<12'` |
| `write_ddl` | Writes DDLs to the specified directory. Directory must exist. | `write_ddl='./MyDDLs'` |
| `read_ddl` | Reads DDLs from the specified directory | `read_ddl='./MyDDLs'` |
| `append` | Activates append mode (details below) | `append='[Hour]>=12'` |
| `useS3`| This defaults to True.  Falls back on individual tables if too small.  Set to false to force using spark writes | `useS3=False` | 

### Predicate Specification

Predicates must run in both Yellowbrick and Databricks, so:

- Column names should be enclosed in `[]`
- Special characters can be escaped using url encoding (for example, `&quot;`)

**Important**: The same predicate is applied on both sides (Databricks and Yellowbrick). Ensure it is correct, especially in append mode.

Examples:

```python
DB2YB(table_pattern=['HourlySummary','HourlyDetail'], where='[hour] between 4 and 10')

DB2YB(table_pattern='.*', limit=1000000)
```

### Append Mode

Append mode is used to insert only new or updated data into existing Yellowbrick tables. Key behaviors:

- Does not drop or recreate the table
- Extracts only data matching the specified WHERE clause from Databricks
- Deletes matching rows in Yellowbrick to prevent duplicates
- Loads the new data

---

## Running as a Workflow (Non-interactively)
To run this non-interactively, you should delete the sample code blocks (3-7) and replace with the DB2YB command most relevant to your workflow.  An example that find the most recent completed hour and appends only that hour of data to Yellowbrick, Below is the code to replace blocks 3-7:

```
from datetime import datetime, timedelta
Hour = datetime.now() - timedelta(hours=1) 
DB2YB(table_patterns=['HourlyDetails'], append=f"[Hour]={Hour.hour}")
```

---

## Notes and Tips

- For large tables, consider increasing the `SPARK_PARTITIONS` constant for better performance.
- Default file format is CSV.
- Always double-check schema and database targeting before running destructive operations.
- Monitor both S3 and Yellowbrick after completion to validate successful ingestion.

---

## Troubleshooting

- **S3 Errors**: Check IAM permissions and AWS region settings.
- **Database Errors**: Verify Yellowbrick IP access rules, login credentials, and schema permissions. Refer to the Yellowbrick VPC Peering guide if needed.
- **Large Table Memory Errors**: Repartition the Spark DataFrame into more pieces.

---
