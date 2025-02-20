# <img src="https://docs.delta.io/latest/_static/delta-lake-white.png" width="150" alt="Delta Lake Logo"></img> Connectors

[![CircleCI](https://circleci.com/gh/delta-io/connectors/tree/master.svg?style=svg)](https://circleci.com/gh/delta-io/connectors/tree/master)

We are building connectors to bring [Delta Lake](https://delta.io) to popular big-data engines outside [Apache Spark](https://spark.apache.org) (e.g., [Apache Hive](https://hive.apache.org/), [Presto](https://prestodb.io/)) and also to common reporting tools like [Microsoft Power BI](https://powerbi.microsoft.com/).

# Introduction

This is the repository for Delta Lake Connectors. It includes a library for querying Delta Lake metadata and connectors to popular big-data engines (e.g., [Apache Hive](https://hive.apache.org/), [Presto](https://prestodb.io/)) and to common reporting tools like [Microsoft Power BI](https://powerbi.microsoft.com/). Please refer to the main [Delta Lake](https://github.com/delta-io/delta) repository if you want to learn more about the Delta Lake project.

# Building

The project is compiled using [SBT](https://www.scala-sbt.org/1.x/docs/Command-Line-Reference.html). It has the following subprojects.

## Delta Standalone Reader
Delta Standalone Reader is a JVM library to read Delta Lake tables. Unlike https://github.com/delta-io/delta, this project doesn't use Spark to read tables and it has only a few transitive dependencies. It can be used by any application that cannot use a Spark cluster.
- To compile the project, run `build/sbt standalone/compile`
- To test the project, run `build/sbt standalone/test`
- To generate the JAR, run `build/sbt standalone/package`

### How to use it
You can add the Delta Standalone Reader library as a dependency using your favorite build tool.

#### Maven
Scala 2.12:
```xml
<dependency>
  <groupId>io.delta</groupId>
  <artifactId>delta-standalone_2.12</artifactId>
  <version>0.2.0</version>
</dependency>
```

Scala 2.11:
```xml
<dependency>
  <groupId>io.delta</groupId>
  <artifactId>delta-standalone_2.11</artifactId>
  <version>0.2.0</version>
</dependency>
```

#### SBT
```
libraryDependencies += "io.delta" %% "delta-standalone" % "0.2.0"
```

See [Delta Standalone Reader](https://github.com/delta-io/connectors/wiki/Delta-Standalone-Reader) for more details.

## Hive connector
This project is a library to make Hive read Delta Lake tables. The project provides a uber JAR `delta-hive-assembly_<scala_version>-0.2.0.jar` to use in Hive. You can use either Scala 2.11 or 2.12. The released JARs are available in the [releases](https://github.com/delta-io/connectors/releases) page. Please download the uber JAR for the corresponding Scala version you would like to use.

You can also use the following instructions to build it as well.

### Build the uber JAR

Please skip this section if you have downloaded the connector JARs.

- To compile the project, run `build/sbt hive/compile`
- To test the project, run `build/sbt hive/test`
- To generate the uber JAR that contains all libraries needed for Hive, run `build/sbt hive/assembly`

The above commands will generate the following JAR:

```
hive/target/scala-2.12/delta-hive-assembly_2.12-0.2.0.jar
```

This uber JAR includes the Hive connector and all its dependencies. They need to be put in Hive’s classpath.

Note: if you would like to build using Scala 2.11, you can run the SBT command `build/sbt "++ 2.11.12 hive/assembly"` to generate the following JAR:

```
hive/target/scala-2.11/delta-hive-assembly_2.11-0.2.0.jar
```

### Setting up Hive

This section describes how to set up Hive to load the Delta Hive connector.

#### Configure Input Formats

Before starting your Hive CLI or running your Hive script, add the following special Hive config to the `hive-site.xml` file. (Its location is `/etc/hive/conf/hive-site.xml` in an EMR cluster).

```xml
<property>
  <name>hive.input.format</name>
  <value>io.delta.hive.HiveInputFormat</value>
</property>
<property>
  <name>hive.tez.input.format</name>
  <value>io.delta.hive.HiveInputFormat</value>
</property>
```

Alternatively, you can also run the following SQL commands in Hive CLI before reading Delta tables to set `io.delta.hive.HiveInputFormat`:

```
SET hive.input.format=io.delta.hive.HiveInputFormat;
SET hive.tez.input.format=io.delta.hive.HiveInputFormat;
```

#### Add Hive uber JAR

The second step is to upload the above uber JAR to the machine that runs Hive. Next, make the JAR accessible to Hive. There are several ways to do this, listed below. To verify that the JAR was properly added, run `LIST JARS;` in the Hive CLI.

- in the Hive CLI, run `ADD JAR <path-to-jar>;`
- add the uber JAR to a folder already pointed to by the `HIVE_AUX_JARS_PATH` environmental variable
- modify the same `hive-site.xml` file as above, and add the following. (Note that this has to be done before you start the Hive CLI)
```xml
<property>
  <name>hive.aux.jars.path</name>
  <value>path_to_uber_jar</value>
</property>
```
- add the path of the uber JAR to Hive’s environment variable, `HIVE_AUX_JARS_PATH`. You can find this environment variable in the `hive-env.sh` file, whose location is `/etc/hive/conf/hive-env.sh` on an EMR cluster. This setting will tell Hive where to find the connector JAR. Ensure you source the script with `source /etc/hive/conf/hive-env.sh`.

### Create a Hive table

After finishing setup, you should be able to create a Delta table in Hive.

Right now the connector supports only EXTERNAL Hive tables. The Delta table must be created using Spark before an external Hive table can reference it.

Here is an example of a CREATE TABLE command that defines an external Hive table pointing to a Delta table on `s3://foo-bucket/bar-dir`.

```SQL
CREATE EXTERNAL TABLE deltaTable(col1 INT, col2 STRING)
STORED BY 'io.delta.hive.DeltaStorageHandler'
LOCATION '/delta/table/path'
```

`io.delta.hive.DeltaStorageHandler` is the class that implements Hive data source APIs. It will know how to load a Delta table and extract its metadata. The table schema in the `CREATE TABLE` statement must be consistent with the underlying Delta metadata. Otherwise, the connector will throw an error to tell you about the inconsistency.

#### Specifying paths in LOCATION
`/delta/table/path` in LOCATION is a normal path. If there is no scheme in the path, it will use the default file system specified in your Hadoop configuration.
You can add an explicit scheme to specify which file system you would like to use, such as `file:///delta/table/path`, `s3://your-s3-bucket/delta/table/path`.

### Frequently asked questions (FAQ)

#### Supported Hive versions
Hive 2.x.

#### Can I use this connector in Apache Spark or Presto?
No. The connector **must** be used with Apache Hive. It doesn't work in other systems, such as Apache Spark or Presto.
- This connector does not provide the support for defining Hive Metastore tables in Apache Spark. It will be added in [Delta Lake core repository](https://github.com/delta-io/delta). It is tracked by the issue https://github.com/delta-io/delta/issues/85.
- This Hive connector does not native connectivity for Presto. But you can generate a manifest file to load a Delta table in Presto. See https://docs.delta.io/latest/presto-integration.html.
- Other system support can be found in https://docs.delta.io/latest/integrations.html.

#### If I create a table using the connector in Hive, can I query it in Apache Spark or Presto?
No. The table created by this connector in Hive cannot be read in any other systems right now. We recommend to create different tables in different systems but point to the same path. Although you need to use different table names to query the same Delta table, the underlying data will be shared by all of systems.

#### Can I write to a Delta table using this connector?
No. The connector doesn't support writing to a Delta table.

#### Do I need to specify the partition columns when creating a Delta table?
No. The partition columns are read from the underlying Delta metadata. The connector will know the partition columns and use this information to do the partition pruning automatically.

#### Why do I need to specify the table schema? Shouldn’t it exist in the underlying Delta table metadata?
Unfortunately, the table schema is a core concept of Hive and Hive needs it before calling the connector.

#### What if I change the underlying Delta table schema in Spark after creating the Hive table?
If the schema in the underlying Delta metadata is not consistent with the schema specified by `CREATE TABLE` statement, the connector will report an error when loading the table and ask you to fix the schema. You must drop the table and recreate it using the new schema. Hive 3.x exposes a new API to allow a data source to hook ALTER TABLE. You will be able to use ALTER TABLE to update a table schema when the connector supports Hive 3.x.

#### Hive has three execution engines, MapReduce, Tez and Spark. Which one does this connector support?
The connector supports MapReduce and Tez. It doesn't support Spark execution engine in Hive.

## sql-delta-import

[sql-delta-import](/sql-delta-import/readme.md) allows for importing data from a JDBC source into a Delta Lake table

## Power BI connector
The connector for [Microsoft Power BI](https://powerbi.microsoft.com/) is basically just a custom Power Query function that allows you to read a Delta Lake table from any file-based [data source supported by Microsoft Power BI](https://docs.microsoft.com/en-us/power-bi/connect-data/desktop-data-sources). Details can be found in the dedicated [README.md](/powerbi/README.md).

# Reporting issues

We use [GitHub Issues](https://github.com/delta-io/connectors/issues) to track community reported issues. You can also [contact](#community) the community for getting answers.

# Contributing

We welcome contributions to Delta Lake Connectors repository. We use [GitHub Pull Requests](https://github.com/delta-io/connectors/pulls) for accepting changes.

# Community

There are two mediums of communication within the Delta Lake community.

- Public Slack Channel
  - [Register here](https://join.slack.com/t/delta-users/shared_invite/enQtNTY1NDg0ODcxOTI1LWJkZGU3ZmQ3MjkzNmY2ZDM0NjNlYjE4MWIzYjg2OWM1OTBmMWIxZTllMjg3ZmJkNjIwZmE1ZTZkMmQ0OTk5ZjA)
  - [Login here](https://delta-users.slack.com/)

- Public [Mailing list](https://groups.google.com/forum/#!forum/delta-users)

# Local Development & Testing
- Before local debugging of `standalone` tests in IntelliJ, run all `standalone` tests using SBT. This helps IntelliJ recognize the golden tables as class resources.
