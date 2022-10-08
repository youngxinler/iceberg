---
title: "Java Quickstart"
url: java-api-quickstart
aliases:
    - "java/quickstart"
menu:
    main:
        parent: "API"
        weight: 100
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance with
 - the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
 -->

# Java API Quickstart

## Create a table

Tables are created using either a [`Catalog`](../../../javadoc/{{% icebergVersion %}}/index.html?org/apache/iceberg/catalog/Catalog.html) or an implementation of the [`Tables`](../../../javadoc/{{% icebergVersion %}}/index.html?org/apache/iceberg/Tables.html) interface.

### Using a Hive catalog

The Hive catalog connects to a Hive metastore to keep track of Iceberg tables.
You can initialize a Hive catalog with a name and some properties.
(see: [Catalog properties](../configuration/#catalog-properties))

**Note:** Currently, `setConf` is always required for hive catalogs, but this will change in the future.

```java
import org.apache.iceberg.hive.HiveCatalog;

HiveCatalog catalog = new HiveCatalog();
catalog.setConf(spark.sparkContext().hadoopConfiguration());  // Configure using Spark's Hadoop configuration

Map <String, String> properties = new HashMap<String, String>();
properties.put("warehouse", "...");
properties.put("uri", "...");

catalog.initialize("hive", properties);
```

The `Catalog` interface defines methods for working with tables, like `createTable`, `loadTable`, `renameTable`, and `dropTable`. `HiveCatalog` implements the `Catalog` interface.

To create a table, pass an `Identifier` and a `Schema` along with other initial metadata:

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;

TableIdentifier name = TableIdentifier.of("logging", "logs");
Table table = catalog.createTable(name, schema, spec);

// or to load an existing table, use the following line
// Table table = catalog.loadTable(name);
```

The logs [schema](#create-a-schema) and [partition spec](#create-a-partition-spec) are created below.


### Using a Hadoop catalog

A Hadoop catalog doesn't need to connect to a Hive MetaStore, but can only be used with HDFS or similar file systems that support atomic rename. Concurrent writes with a Hadoop catalog are not safe with a local FS or S3. To create a Hadoop catalog:

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.iceberg.hadoop.HadoopCatalog;

Configuration conf = new Configuration();
String warehousePath = "hdfs://host:8020/warehouse_path";
HadoopCatalog catalog = new HadoopCatalog(conf, warehousePath);
```

Like the Hive catalog, `HadoopCatalog` implements `Catalog`, so it also has methods for working with tables, like `createTable`, `loadTable`, and `dropTable`.
                                                                                       
This example creates a table with Hadoop catalog:

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;

TableIdentifier name = TableIdentifier.of("logging", "logs");
Table table = catalog.createTable(name, schema, spec);

// or to load an existing table, use the following line
// Table table = catalog.loadTable(name);
```

The logs [schema](#create-a-schema) and [partition spec](#create-a-partition-spec) are created below.


### Using Hadoop tables

Iceberg also supports tables that are stored in a directory in HDFS. Concurrent writes with a Hadoop tables are not safe when stored in the local FS or S3. Directory tables don't support all catalog operations, like rename, so they use the `Tables` interface instead of `Catalog`.

To create a table in HDFS, use `HadoopTables`:

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.iceberg.hadoop.HadoopTables;
import org.apache.iceberg.Table;

Configuration conf = new Configuration();
HadoopTables tables = new HadoopTables(conf);
Table table = tables.create(schema, spec, table_location);

// or to load an existing table, use the following line
// Table table = tables.load(table_location);
```

{{< hint danger >}}
Hadoop tables shouldn't be used with file systems that do not support atomic rename. Iceberg relies on rename to synchronize concurrent commits for directory tables.
{{< /hint >}}

### Tables in Spark

Spark uses both `HiveCatalog` and `HadoopTables` to load tables. Hive is used when the identifier passed to `load` or `save` is not a path, otherwise Spark assumes it is a path-based table.

To read and write to tables from Spark see:

* [SQL queries in Spark](../spark-queries#querying-with-sql)
* [`INSERT INTO` in Spark](../spark-writes#insert-into)
* [`MERGE INTO` in Spark](../spark-writes#merge-into)


## Schemas

### Create a schema

This example creates a schema for a `logs` table:

```java
import org.apache.iceberg.Schema;
import org.apache.iceberg.types.Types;

Schema schema = new Schema(
      Types.NestedField.required(1, "level", Types.StringType.get()),
      Types.NestedField.required(2, "event_time", Types.TimestampType.withZone()),
      Types.NestedField.required(3, "message", Types.StringType.get()),
      Types.NestedField.optional(4, "call_stack", Types.ListType.ofRequired(5, Types.StringType.get()))
    );
```

When using the Iceberg API directly, type IDs are required. Conversions from other schema formats, like Spark, Avro, and Parquet will automatically assign new IDs.

When a table is created, all IDs in the schema are re-assigned to ensure uniqueness.

### Convert a schema from Avro

To create an Iceberg schema from an existing Avro schema, use converters in `AvroSchemaUtil`:

```java
import org.apache.avro.Schema;
import org.apache.avro.Schema.Parser;
import org.apache.iceberg.avro.AvroSchemaUtil;

Schema avroSchema = new Parser().parse("{\"type\": \"record\" , ... }");
Schema icebergSchema = AvroSchemaUtil.toIceberg(avroSchema);
```

### Convert a schema from Spark

To create an Iceberg schema from an existing table, use converters in `SparkSchemaUtil`:

```java
import org.apache.iceberg.spark.SparkSchemaUtil;

Schema schema = SparkSchemaUtil.schemaForTable(sparkSession, table_name);
```

## Partitioning

### Create a partition spec

Partition specs describe how Iceberg should group records into data files. Partition specs are created for a table's schema using a builder.

This example creates a partition spec for the `logs` table that partitions records by the hour of the log event's timestamp and by log level:

```java
import org.apache.iceberg.PartitionSpec;

PartitionSpec spec = PartitionSpec.builderFor(schema)
      .hour("event_time")
      .identity("level")
      .build();
```

For more information on the different partition transforms that Iceberg offers, visit [this page](../../../spec#partitioning).


## WriteRecord

### PartitionedTable write

Use iceberg api write record into PartitionedTable

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.iceberg.*;
import org.apache.iceberg.catalog.Catalog;
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.data.GenericAppenderFactory;
import org.apache.iceberg.data.GenericRecord;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.hadoop.HadoopCatalog;
import org.apache.iceberg.io.OutputFileFactory;
import org.apache.iceberg.io.PartitionedFanoutWriter;
import org.junit.Test;

import java.util.Arrays;
import java.util.List;
import java.util.Random;

Configuration configuration = new Configuration();
Catalog hadoopCatalog = new HadoopCatalog(configuration, icebergWareHousePath);

// Load table from catalog
TableIdentifier name = TableIdentifier.of("logging", "logs_partitioned");
Table table = hadoopCatalog.loadTable(name);

//  assume logs_partitioned table schema and spec was created
//  Schema schema = new Schema(
//  Types.NestedField.required(1, "level", Types.StringType.get()),
//  Types.NestedField.required(2, "event_time", Types.LongType.get()),
//  Types.NestedField.required(3, "message", Types.StringType.get())
//  );
        
GenericAppenderFactory appenderFactory = new GenericAppenderFactory(table.schema());
int partitionId = 1, taskId = 1;
OutputFileFactory outputFileFactory = OutputFileFactory.builderFor(table, partitionId, taskId).format(FileFormat.PARQUET).build();
final PartitionKey partitionKey = new PartitionKey(table.spec(), table.spec().schema());

// partitionedFanoutWriter will auto partitioned record and create the partitioned writer
PartitionedFanoutWriter<Record> partitionedFanoutWriter = new PartitionedFanoutWriter<Record>(table.spec(), FileFormat.PARQUET, appenderFactory, outputFileFactory, table.io(), TARGET_FILE_SIZE_IN_BYTES) {
    @Override
    protected PartitionKey partition(Record record) {
        partitionKey.partition(record);
        return partitionKey;
    }
};

Random random = new Random();
List<String> levels = Arrays.asList("info", "debug", "error", "warn");
GenericRecord genericRecord = GenericRecord.create(table.schema());

// assume write 1000 records
for (int i = 0; i < 1000; i++) {
    GenericRecord record = genericRecord.copy();
    record.setField("level",  levels.get(random.nextInt(levels.size())));
    record.setField("event_time", System.currentTimeMillis());
    record.setField("message", "Iceberg is a great table format");
    partitionedFanoutWriter.write(record);
}

AppendFiles appendFiles = table.newAppend();
// submit datafiles to the table
Arrays.stream(partitionedFanoutWriter.dataFiles()).forEach(appendFiles::appendFile);

// submit snapshot
Snapshot newSnapshot = appendFiles.apply();
appendFiles.commit();
```