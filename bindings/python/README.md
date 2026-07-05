<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one
  ~ or more contributor license agreements.  See the NOTICE file
  ~ distributed with this work for additional information
  ~ regarding copyright ownership.  The ASF licenses this file
  ~ to you under the Apache License, Version 2.0 (the
  ~ "License"); you may not use this file except in compliance
  ~ with the License.  You may obtain a copy of the License at
  ~
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing,
  ~ software distributed under the License is distributed on an
  ~ "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  ~ KIND, either express or implied.  See the License for the
  ~ specific language governing permissions and limitations
  ~ under the License.
-->

# PyPaimon Rust

This project builds the Rust-powered core for [PyPaimon](https://paimon.apache.org/docs/master/pypaimon/overview/) while also providing DataFusion integration for querying Paimon tables.

## Usage

```python
import pyarrow as pa
from pypaimon_rust.datafusion import SQLContext

# Create a SQL context and register a Paimon catalog
ctx = SQLContext()
ctx.register_catalog("paimon", {"warehouse": "/tmp/paimon-warehouse"})

# Create a table and insert data
ctx.sql("CREATE SCHEMA paimon.my_db")
ctx.sql("CREATE TABLE paimon.my_db.users (id INT, name STRING, PRIMARY KEY (id))")
ctx.sql("INSERT INTO paimon.my_db.users VALUES (1, 'alice'), (2, 'bob')")

# Query data
batches = ctx.sql("SELECT id, name FROM paimon.my_db.users ORDER BY id")

# Register a temporary table from a PyArrow RecordBatch
batch = pa.record_batch([[1, 2], ["alice", "bob"]], names=["id", "name"])
ctx.register_batch("paimon.default.my_temp", batch)
batches = ctx.sql("SELECT * FROM paimon.default.my_temp")

# Drop it via SQL when no longer needed
ctx.sql("DROP TEMPORARY TABLE paimon.default.my_temp")
```

For the full SQL reference, see the [SQL Integration docs](https://paimon.apache.org/docs/master/sql/).

### Native Read / Write

Beyond SQL, you can use the lower-level read and write APIs directly from Python.
Time travel is supported via the `options` dict on `new_read_builder`.

```python
from pypaimon_rust.datafusion import PaimonCatalog

catalog = PaimonCatalog({"warehouse": "/tmp/paimon-warehouse"})

# Table must already exist — create it via SQL, the REST API, or another client
table = catalog.get_table("my_db.my_table")

# Time travel: pass timestamp-millis, version, snapshot-id, or tag-name
reader = (
    table.new_read_builder({"scan.timestamp-millis": "1718208000000"})
    .with_projection(["id", "name"])
    .with_limit(100)
    .new_read()
)

# Scan → Read: plan splits first, then read batches
scan = table.new_read_builder().new_scan()
plan = scan.plan()
batches = reader.read(plan.splits())

# Write: build → write Arrow batches → commit
write_builder = table.new_write_builder()
writer = write_builder.new_write()
writer.write_arrow(batch)
commit_messages = writer.prepare_commit()

write_builder.new_commit().commit(commit_messages)
```

## Setup

Install [uv](https://docs.astral.sh/uv/getting-started/installation/):

```shell
pip install uv
```

Set up the development environment:

```shell
make install
```

## Build

```shell
make build
```

## Test

Python integration tests expect the shared Paimon test warehouse to be prepared
first from the repository root:

```shell
make docker-up
cd bindings/python
```

```shell
make test
```
