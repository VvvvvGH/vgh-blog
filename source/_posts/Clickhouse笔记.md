---
title: Clickhouse笔记
date: 2024-05-06 11:27:25
tags:
---

# Clickhouse笔记

1. 创建包含 JSON 表的时候，需要使用实验功能

   ```sql
   SET allow_experimental_object_type=1;
   CREATE table rc_field_json(fieldData JSON,event String, timestamp DATETIME64) ENGINE = MergeTree ORDER BY (event, timestamp)
   ```

2. TCP 客户端不支持  JSON 

   ```
   Not implemented for SerializationObject: While executing BinaryRowInputFormat
   ```

3. 主键索引指南

   https://clickhouse.com/docs/en/optimize/sparse-primary-indexes

4. 支持字段、行、表级别的 TTL 设置

   https://clickhouse.com/docs/en/guides/developer/ttl

   TTL事件的触发条件：

   * table merge
   * `merge_with_ttl_timeout`: the minimum delay in seconds before repeating a merge with delete TTL. The default is 14400 seconds (4 hours).
   * `merge_with_recompression_ttl_timeout`: the minimum delay in seconds before repeating a merge with recompression TTL (rules that roll up data before deleting). Default value: 14400 seconds (4 hours).



5. 大量小批量插入的优化方案

   https://clickhouse.com/docs/en/optimize/asynchronous-insertsEnabling asynchronous inserts[](https://clickhouse.com/docs/en/optimize/asynchronous-inserts#enabling-asynchronous-inserts)

   Asynchronous inserts can be enabled for a particular user, or for a specific query:

   - Enabling asynchronous inserts at the user level. This example uses the user `default`, if you create a different user then substitute that username:

     ```sql
     ALTER USER default SETTINGS async_insert = 1
     ```

     

   - You can specify the asynchronous insert settings by using the SETTINGS clause of insert queries:

     ```sql
     INSERT INTO YourTable SETTINGS async_insert=1, wait_for_async_insert=1 VALUES (...)
     ```

     

   - You can also specify asynchronous insert settings as connection parameters when using a ClickHouse programming language client.

     As an example, this is how you can do that within a JDBC connection string when you use the ClickHouse Java JDBC driver for connecting to ClickHouse Cloud :

     ```bash
     "jdbc:ch://HOST.clickhouse.cloud:8443/?user=default&password=PASSWORD&ssl=true&custom_http_params=async_insert=1,wait_for_async_insert=1"
     ```

     

     Our strong recommendation is to use async_insert=1,wait_for_async_insert=1 if using asynchronous inserts. Using wait_for_async_insert=0 is very risky because your INSERT client may not be aware if there are errors, and also can cause potential overload if your client continues to write quickly in a situation where the ClickHouse server needs to slow down the writes and create some backpressure in order to ensure reliability of the service.

6. 物化视图

   物化视图是一种在写入数据时对数据进行预处理和聚合的方法，可以提高查询性能。
   物化视图可以级联，即一个物化视图的目标表可以作为另一个物化视图的源表。
   创建物化视图需要先创建源表和目标表，然后指定物化视图的查询逻辑。
   源表可以使用Null表引擎来节省存储空间，因为源数据不需要保留。
   目标表可以使用AggregatingMergeTree或SummingMergeTree等表引擎来支持增量聚合和合并。
   物化视图可以使用sumState、sumMerge等聚合函数来计算聚合状态和结果。
   物化视图有一些限制，例如不能使用ORDER BY、LIMIT、DISTINCT等子句，不能使用JOIN、UNION ALL等操作符，不能使用窗口函数等。

7. 查询表存储空间

   ```sql
   SELECT
       table AS `表名`,
       sum(rows) AS `总行数`,
       formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
       formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
       round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
   FROM system.parts
   WHERE table IN ('event_db_data')
   GROUP BY table
   ```

   

8. 内存需求

   https://clickhouse.com/docs/ru/operations/tips

   
## 写入延迟优化

### 1. 优化小批量写入性能

1. **合并写入**：尝试将多个小批量数据合并为一个较大的批量，然后一次性写入 Clickhouse，以减少写入操作的次数和提高写入速度。

2. **使用缓冲表**：使用 `Buffer` 引擎创建一个缓冲表，可以在内存中缓存数据批，当达到特定条件时自动将数据批导入实际表。这样可以提高写入速度，但需要注意内存占用情况。

 `Buffer` 表引擎用于创建缓冲表，它接收9个参数，这些参数控制缓冲的行为和性能。下面是每个参数的详细解释：

   1. `database` (ffpay_risk_control): 目标数据库的名称，缓冲表将把数据刷入这个数据库中的目标表。
   2. `target_table` (strategy_execution_log_actual): 目标表的名称，缓冲表将把数据刷入这个表中。
   3. `num_layers` (16): 缓冲区的层数，每层都有独立的条件，当满足条件时，会将数据从当前层向下一层传递，最后刷入目标表。较高的层数可以增加缓冲区的容量和写入性能，但同时会消耗更多的内存。
   4. `min_time` (10): 最小生存时间，单位为秒。数据在当前层至少需要存活这么多时间，才有可能向下一层传递。这个参数可以控制数据在缓冲区内的最小生存时间。
   5. `max_time` (100): 最大生存时间，单位为秒。数据在当前层达到这个时间后，必须向下一层传递。这个参数可以控制数据在缓冲区内的最大生存时间。
   6. `min_rows` (10000): 最小行数。只有当当前层的数据行数达到这个值时，数据才有可能向下一层传递。这个参数可以控制数据满足的最小行数条件。
   7. `max_rows` (1000000): 最大行数。当当前层的数据行数达到这个值时，数据必须向下一层传递。这个参数可以控制数据满足的最大行数条件。
   8. `min_bytes` (10000000): 最小字节数。只有当当前层的数据字节数达到这个值时，数据才有可能向下一层传递。这个参数可以控制数据满足的最小字节数条件。
   9. `max_bytes` (100000000): 最大字节数。当当前层的数据字节数达到这个值时，数据必须向下一层传递。这个参数可以控制数据满足的最大字节数条件。

   当任意一个条件（生存时间、行数或字节数）满足当前层的最大值时，数据将向下一层传递，最终在满足最后一层的条件时会刷入目标表。如果数据在当前层的生存时间已经超过最小值，但未达到最大值，此时如果满足行数或字节数的最小条件，数据也会向下一层传递。

   这些参数需要根据您的实际应用场景和性能要求进行调整。一般来说，调整这些参数可以在保证写入性能和查询实时性的同时，平衡内存和CPU资源的消耗。

   

3. **异步写入**：将批量数据写入 Clickhouse 的操作放到后台执行，避免阻塞后续的写入操作。

### 2. 提高并发查询性能

1. **索引优化**：创建更适合查询场景的索引，如 `PRIMARY KEY`、`ORDER BY` 以及 `SAMPLE BY`。优化索引可以提高查询性能并减少数据扫描量。
2. **分区优化**：合理划分分区，根据业务场景和查询需求，使用 `PARTITION BY` 语句创建分区。通过减少查询涉及的分区，提高查询性能。
3. **数据分布和复制**：配置 `Replicated*` 表引擎和使用 `Distributed` 表引擎，以保证数据在多个节点上均匀分布并实现负载均衡。这有助于提高查询性能和可用性。
4. **资源限制调整**：根据需求调整 Clickhouse 的资源限制参数 (如内存、CPU、磁盘 I/O 等)，以满足不同场景下的性能需求。
5. **查询优化**：优化查询语句，避免全表扫描和使用耗时较长的函数。使用 `PREWHERE` 子句对常用过滤条件进行预过滤，减少数据传输和计算量。

通过实施以上优化策略，可以提高 Clickhouse 在实时数据收集与分析场景下的小批量写入速度和并发查询性能。