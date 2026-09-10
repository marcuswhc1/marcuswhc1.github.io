---
title: Connecting to Oracle Database using Python and finding server details using tnsping
date:  2026-07-08 22:00:00 +0800
categories: [Knowledge, Python]
tags: [python, oracle, documentation, knowledge]
description: A guide on connecting to Oracle Database using the oracledb library in Python and how to find full server connection details using tnsping.
---

## Imports
In order to connect to Oracle Database using Python, the oracledb library will need to be installed first as it is a third-party library.
The oracledb library is Oracle's official Python driver for Oracle Database, supporting both Thin and Thick connection modes. Thin mode is preferred as it does not require Oracle Client libraries to be installed, making it suitable for containerised environments such as Red Hat OpenShift pods.
The library will be installed using pip, the standard package manager for Python, through the command prompt.

```shell
pip install oracledb
```
After installation, import the oracledb library alongside the os library to retrieve credentials from environment variables.
```python
import oracledb
import os
```

## Retrieving credentials from environment variables
Store Oracle credentials as environment variables instead of hardcoding them into the code. This is especially important in containerised deployments where secrets are injected at pod startup.
```python
#--------------------------
# Oracle Database Credentials
#--------------------------
oracle_user     = os.environ.get('ORACLE_USER')
oracle_password = os.environ.get('ORACLE_PASSWORD')
oracle_dsn      = os.environ.get('ORACLE_DSN')
```

## Finding full server connection details using tnsping
Before connecting, the full DSN (Data Source Name) string is required in the following format:
```
hostname:port/service_name
```
To find these details, run the following command in the Windows Command Prompt, replacing `your_alias` with your Oracle TNS alias:
```shell
tnsping your_alias
```
The output will return the full connection details:
```
Used LDAP adapter to resolve the alias
Attempting to contact (description=(address_list=(address=(protocol=tcp)(host=your-db-host.domain.com)(port=1522)))(connect_data=(sid=yourdbname)))
OK (390 msec)
```
Extract the following from the output:
```
host = your-db-host.domain.com   # the full database hostname
port = 1522                      # note: Oracle does not always use default port 1521
sid  = yourdbname                # used as service name in DSN string
```
Your full DSN string will be:
```
your-db-host.domain.com:1522/yourdbname
```

Alternatively, run the following SQL query in PL/SQL Developer to retrieve connection details:
```sql
SELECT
    SYS_CONTEXT('USERENV', 'DB_NAME')      AS db_name,
    SYS_CONTEXT('USERENV', 'SERVICE_NAME') AS service_name,
    SYS_CONTEXT('USERENV', 'SERVER_HOST')  AS host
FROM dual;
```

## Code
Create a function to establish a connection to Oracle Database using Thin mode.
```python
#--------------------------------------
# Function to connect to Oracle Database
#--------------------------------------
def get_oracle_connection():
    try:
        connection = oracledb.connect(
            user=oracle_user,
            password=oracle_password,
            dsn=oracle_dsn      # format: hostname:port/service_name
        )
        return connection
    except oracledb.DatabaseError as e:
        print(f"Error connecting to Oracle Database: {e}")
        return None
```

## Use case
The example below demonstrates how to query data from Oracle Database into a pandas DataFrame.
```python
import pandas as pd

#--------------------------------------
# Function to load data from Oracle DB
#--------------------------------------
def load_data():
    try:
        connection = get_oracle_connection()

        query = """
            SELECT *
            FROM your_schema.your_table
        """

        with connection:
            df = pd.read_sql(query, connection)

        return df

    except oracledb.DatabaseError as e:
        print(f"Error occurred while connecting to Oracle database: {e}")
        return None
```

## Notes
- Always use **Thin mode** (default in oracledb) in containerised environments - it does not require Oracle Client libraries or `tnsnames.ora` files.
- The default Oracle port is `1521`, however some databases may use a different port (e.g. `1522`). Always verify using `tnsping`.
- Never hardcode credentials directly in code - use environment variables or secrets management.
````
