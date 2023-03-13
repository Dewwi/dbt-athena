[![pypi](https://badge.fury.io/py/dbt-athena-community.svg)](https://pypi.org/project/dbt-athena-community/)
[![Imports: isort](https://img.shields.io/badge/%20imports-isort-%231674b1?style=flat&labelColor=ef8336)](https://pycqa.github.io/isort/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Stats: pepy](https://pepy.tech/badge/dbt-athena-community/month)](https://pepy.tech/project/dbt-athena-community)

# dbt-athena

* Supports dbt version `1.4.*`
* Supports [Seeds][seeds]
* Correctly detects views and their columns
* Supports [table materialization][table]
  * [Iceberg tables][athena-iceberg] is supported **only with Athena Engine v3** and **a unique table location**
  (see table location section below)
  * Hive tables is supported by both Athena engines.
* Supports [incremental models][incremental]
  * On iceberg tables :
    * Support the use of `unique_key` only with the `merge` strategy
    * Support the `append` strategy
  * On Hive tables :
    * Support two incremental update strategies: `insert_overwrite` and `append`
    * Does **not** support the use of `unique_key`
* Supports [snapshots][snapshots]
* Does not support [Python models][python-models]
* Does not support [persist docs][persist-docs] for views

[seeds]: https://docs.getdbt.com/docs/building-a-dbt-project/seeds
[incremental]: https://docs.getdbt.com/docs/build/incremental-models
[table]: https://docs.getdbt.com/docs/build/materializations#table
[python-models]: https://docs.getdbt.com/docs/build/python-models#configuring-python-models
[athena-iceberg]: https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html
[snapshots]: https://docs.getdbt.com/docs/build/snapshots
[persist-docs]: https://docs.getdbt.com/reference/resource-configs/persist_docs

### Installation

* `pip install dbt-athena-community`
* Or `pip install git+https://github.com/dbt-athena/dbt-athena.git`

### Prerequisites

To start, you will need an S3 bucket, for instance `my-bucket` and an Athena database:

```sql
CREATE DATABASE IF NOT EXISTS analytics_dev
COMMENT 'Analytics models generated by dbt (development)'
LOCATION 's3://my-bucket/'
WITH DBPROPERTIES ('creator'='Foo Bar', 'email'='foo@bar.com');
```

Notes:
- Take note of your AWS region code (e.g. `us-west-2` or `eu-west-2`, etc.).
- You can also use [AWS Glue](https://docs.aws.amazon.com/athena/latest/ug/glue-athena.html) to create and manage Athena databases.

### Credentials

This plugin does not accept any credentials directly. Instead, [credentials are determined automatically](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) based on `aws cli`/`boto3` conventions and
stored login info. You can configure the AWS profile name to use via `aws_profile_name`. Checkout DBT profile configuration below for details.

### Configuring your profile

A dbt profile can be configured to run against AWS Athena using the following configuration:

| Option           | Description                                                                    | Required?   | Example               |
|------------------|--------------------------------------------------------------------------------|-------------|-----------------------|
| s3_staging_dir   | S3 location to store Athena query results and metadata                         | Required    | `s3://bucket/dbt/`    |
| s3_data_dir      | Prefix for storing tables, if different from the connection's `s3_staging_dir` | Optional    | `s3://bucket2/dbt/`   |
| s3_data_naming   | How to generate table paths in `s3_data_dir`                                   | Optional    | `schema_table_unique` |
| region_name      | AWS region of your Athena instance                                             | Required    | `eu-west-1`           |
| schema           | Specify the schema (Athena database) to build models into (lowercase **only**) | Required    | `dbt`                 |
| database         | Specify the database (Data catalog) to build models into (lowercase **only**)  | Required    | `awsdatacatalog`      |
| poll_interval    | Interval in seconds to use for polling the status of query results in Athena   | Optional    | `5`                   |
| aws_profile_name | Profile to use from your AWS shared credentials file.                          | Optional    | `my-profile`          |
| work_group       | Identifier of Athena workgroup                                                 | Optional    | `my-custom-workgroup` |
| num_retries      | Number of times to retry a failing query                                       | Optional    | `3`                   |


**Example profiles.yml entry:**
```yaml
athena:
  target: dev
  outputs:
    dev:
      type: athena
      s3_staging_dir: s3://athena-query-results/dbt/
      s3_data_dir: s3://your_s3_bucket/dbt/
      s3_data_naming: schema_table
      region_name: eu-west-1
      schema: dbt
      database: awsdatacatalog
      aws_profile_name: my-profile
      work_group: my-workgroup
```

_Additional information_
* `threads` is supported
* `database` and `catalog` can be used interchangeably

### Models

#### Table Configuration

* `external_location` (`default=none`)
  * If set, the full S3 path in which the table will be saved. (Does not work with Iceberg table).
* `partitioned_by` (`default=none`)
  * An array list of columns by which the table will be partitioned
  * Limited to creation of 100 partitions (_currently_)
* `bucketed_by` (`default=none`)
  * An array list of columns to bucket data, ignored if using Iceberg
* `bucket_count` (`default=none`)
  * The number of buckets for bucketing your data, ignored if using Iceberg
* `table_type` (`default='hive'`)
  * The type of table
  * Supports `hive` or `iceberg`
* `format` (`default='parquet'`)
  * The data format for the table
  * Supports `ORC`, `PARQUET`, `AVRO`, `JSON`, `TEXTFILE`
* `write_compression` (`default=none`)
  * The compression type to use for any storage format that allows compression to be specified. To see which options are available, check out [CREATE TABLE AS][create-table-as]
* `field_delimiter` (`default=none`)
  * Custom field delimiter, for when format is set to `TEXTFILE`
* `table_properties`: table properties to add to the table, valid for Iceberg only

#### Table location

The location in which a table is saved is determined by:

1. If `external_location` is defined, that value is used.
2. If `s3_data_dir` is defined, the path is determined by that and `s3_data_naming`
3. If `s3_data_dir` is not defined data is stored under `s3_staging_dir/tables/`

Here all the options available for `s3_data_naming`:
* `uuid`: `{s3_data_dir}/{uuid4()}/`
* `table_table`: `{s3_data_dir}/{table}/`
* `table_unique`: `{s3_data_dir}/{table}/{uuid4()}/`
* `schema_table`: `{s3_data_dir}/{schema}/{table}/`
* `s3_data_naming=schema_table_unique`: `{s3_data_dir}/{schema}/{table}/{uuid4()}/`

It's possible to set the `s3_data_naming` globally in the target profile, or overwrite the value in the table config,
or setting up the value for groups of model in dbt_project.yml.

> Note: when using a work group with a default output location configured, `s3_data_naming` and any configured buckets are ignored and the location configured in the work group is used.


#### Incremental models

Support for [incremental models](https://docs.getdbt.com/docs/build/incremental-models).

These strategies are supported:

* `insert_overwrite` (default): The insert overwrite strategy deletes the overlapping partitions from the destination
table, and then inserts the new records from the source. This strategy depends on the `partitioned_by` keyword! If no
partitions are defined, dbt will fall back to the `append` strategy.
* `append`: Insert new records without updating, deleting or overwriting any existing data. There might be duplicate
data (e.g. great for log or historical data).
* `merge`: Conditionally updates, deletes, or inserts rows into an Iceberg table. Used in combination with `unique_key`.
Only available when using Iceberg.

#### On schema change

`on_schema_change` is an option to reflect changes of schema in incremental models.
The following options are supported:
* `ignore` (default)
* `fail`
* `append_new_columns`
* `sync_all_columns`

In detail, please refer to [dbt docs](https://docs.getdbt.com/docs/build/incremental-models#what-if-the-columns-of-my-incremental-model-change).


#### Iceberg

The adapter supports table materialization for Iceberg.

To get started just add this as your model:
```
{{ config(
    materialized='table',
    table_type='iceberg',
    format='parquet',
    partitioned_by=['bucket(user_id, 5)'],
    table_properties={
    	'optimize_rewrite_delete_file_threshold': '2'
    	}
) }}

SELECT
	'A' AS user_id,
	'pi' AS name,
	'active' AS status,
	17.89 AS cost,
	1 AS quantity,
	100000000 AS quantity_big,
	current_date AS my_date
```

Iceberg supports bucketing as hidden partitions, therefore use the `partitioned_by` config to add specific bucketing conditions.

Iceberg supports several table formats for data : `PARQUET`, `AVRO` and `ORC`.

It is possible to use iceberg in an incremental fashion, specifically 2 strategies are supported:
* `append`: new records are appended to the table, this can lead to duplicates
* `merge`: must be used in combination with `unique_key` and it's only available with Engine version 3.
   It performs an upsert, new record are added, and record already existing are updated

#### High available table materialization
The current implementation of the table materialization can lead to downtime, as target table is dropped and re-created.
To have the less destructive behavior it's possible to use `table='table_hive_ha'` materialization.
**table_hive_ha** leverage the table versions feature of glue catalog, creating a tmp table and swapping
the target table to the location of the tmp table.
This materialization is only available for `table_type=hive` and requires using unique locations.

```
{{ config(
    materialized='table_hive_ha',
    format='parquet',
    partition_by=['status'],
    s3_data_naming='table_unique'
) }}


select
  'a' as user_id,
  'pi' as user_name,
  'active' as status
union all
select
  'b' as user_id,
  'sh' as user_name,
  'disabled' as status
```

By default, the materialization keeps the last 4 table versions, you can change it that setting `versions_to_keep`.

##### Known issues
* When swapping from a table with partitions to a table without (and the other way around), there could be a little downtime.
  In case high performances are needed consider bucketing instead of partitions
* By default, Glue "duplicate" the versions internally, so the last 2 versions of a table point to the same location
* It's recommended to have versions_to_keep>= 4, as this will avoid to have the older location removed
* The macro athena__end_of_time needs to be overwritten by the user if using Athena v3 since it requires a precision parameter for timestamps


### Snapshots

The adapter supports snapshot materialization. It supports both timestamp and check strategy. To create a snapshot create a snapshot file in the snapshots directory. If directory does not exist create one.

#### Timestamp strategy

To use the timestamp strategy refer to the [dbt docs](https://docs.getdbt.com/docs/build/snapshots#timestamp-strategy-recommended)

#### Check strategy

To use the check strategy refer to the [dbt docs](https://docs.getdbt.com/docs/build/snapshots#check-strategy)

#### Hard-deletes

The materialization also supports invalidating hard deletes. Check the [docs](https://docs.getdbt.com/docs/build/snapshots#hard-deletes-opt-in) to understand usage.


### Known issues

* Incremental Iceberg models - Sync all columns on schema change can't remove columns used as partitioning.
The only way, from a dbt perspective, is to do a full-refresh of the incremental model.

* Tables, schemas and database should only be lowercase

* In order to avoid potential conflicts, make sure [`dbt-athena-adapter`](https://github.com/Tomme/dbt-athena) is not installed in the target environment.
  See https://github.com/dbt-athena/dbt-athena/issues/103 for more details.

* Snapshot does not support dropping columns from the source table. If you drop a column make sure to drop the column from the snapshot as well. Another workaround is to NULL the column in the snapshot definition to preserve history

### Contributing

This connector works with Python from 3.7 to 3.11.

#### Getting started

In order to start developing on this adapter clone the repo and run this make command (see [Makefile](Makefile)) :

```bash
make setup
```

It will :
1. Install all dependencies.
2. Install pre-commit hooks.
3. Generate your `.env` file

Next, adjust `.env` file by configuring the environment variables to match your Athena development environment.

#### Running tests

We have 2 different types of testing:
* **unit testing**: you can run this type of tests running `make unit_test`
* **functional testing**: you must have an AWS account with Athena setup in order to launch this type of tests and have a `.env` file in place with the right values.
  You can run this type of tests running `make functional_test`


All type of tests can be run using `make`:
```bash
make test
```

#### Pull Request

* Create a commit with your changes and push them to a
  [fork](https://docs.github.com/en/get-started/quickstart/fork-a-repo).
* Create a [pull request on
  Github](https://docs.github.com/en/github/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork).
* Pull request title and message (and PR title and description) must adhere to
  [conventionalcommits](https://www.conventionalcommits.org).
* Pull request body should describe _motivation_.

### Helpful Resources

* [Athena CREATE TABLE AS](https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html)
* [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core)
* [laughingman7743/PyAthena](https://github.com/laughingman7743/PyAthena)
