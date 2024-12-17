---
title: "DELETE via tombstone column"
linkTitle: "DELETE via tombstone column"
description: >
    DELETE via tombstone column
---

This article provides an overview of the different methods to handle row deletion in ClickHouse, using tombstone columns and ALTER UPDATE or DELETE. The goal is to highlight the performance impacts of different techniques and storage settings, including a scenario using S3 for remote storage.

1. Creating a Test Table
We will start by creating a simple MergeTree table with a tombstone column (is_active) to track active rows:

```sql
CREATE TABLE test_delete
(
    `key` UInt32,
    `ts` UInt32,
    `value_a` String,
    `value_b` String,
    `value_c` String,
    `is_active` UInt8 DEFAULT 1
)
ENGINE = MergeTree
ORDER BY key;
```
2. Inserting Data
Insert sample data into the table:
```sql
INSERT INTO test_delete (key, ts, value_a, value_b, value_c) SELECT
    number,
    1,
    concat('some_looong_string', toString(number)),
    concat('another_long_str', toString(number)),
    concat('string', toString(number))
FROM numbers(10000000);


INSERT INTO test_delete (key, ts, value_a, value_b, value_c) VALUES (400000, 2, 'totally different string', 'another totally different string', 'last string');
```
3. Querying the Data
To verify the inserted data:
```sql
SELECT *
FROM test_delete
WHERE key = 400000;

┌────key─┬─ts─┬─value_a──────────────────┬─value_b──────────────────────────┬─value_c─────┬─is_active─┐
│ 400000 │  2 │ totally different string │ another totally different string │ last string │         1 │
└────────┴────┴──────────────────────────┴──────────────────────────────────┴─────────────┴───────────┘
┌────key─┬─ts─┬─value_a──────────────────┬─value_b────────────────┬─value_c──────┬─is_active─┐
│ 400000 │  1 │ some_looong_string400000 │ another_long_str400000 │ string400000 │         1 │
└────────┴────┴──────────────────────────┴────────────────────────┴──────────────┴───────────┘
```
This should return two rows with different ts values.

4. Soft Deletion Using ALTER UPDATE
Instead of deleting a row, you can mark it as inactive by setting is_active to 0:
```sql

SET mutations_sync = 2;

ALTER TABLE test_delete
    UPDATE is_active = 0 WHERE (key = 400000) AND (ts = 1);
Ok.

0 rows in set. Elapsed: 0.058 sec.
```
After updating, you can filter out inactive rows:
```sql
SELECT *
FROM test_delete
WHERE (key = 400000) AND is_active=0;

┌────key─┬─ts─┬─value_a──────────────────┬─value_b────────────────┬─value_c──────┬─is_active─┐
│ 400000 │  1 │ some_looong_string400000 │ another_long_str400000 │ string400000 │         0 │
└────────┴────┴──────────────────────────┴────────────────────────┴──────────────┴───────────┘
```
5. Hard Deletion Using ALTER DELETE
If you need to completely remove a row from the table, you can use ALTER DELETE:
```sql
ALTER TABLE test_delete
    DELETE WHERE (key = 400000) AND (ts = 1);

Ok.

0 rows in set. Elapsed: 1.101 sec. -- 20 times slower!!!
```
However, this operation is significantly slower compared to the ALTER UPDATE approach. For example:

ALTER DELETE: Takes around 1.1 seconds
ALTER UPDATE: Only 0.05 seconds

The reason for this difference is that DELETE modifies the physical data structure, while UPDATE merely changes a column value.

```sql
SELECT *
FROM test_delete
WHERE key = 400000;

┌────key─┬─ts─┬─value_a──────────────────┬─value_b──────────────────────────┬─value_c─────┬─is_active─┐
│ 400000 │  2 │ totally different string │ another totally different string │ last string │         1 │
└────────┴────┴──────────────────────────┴──────────────────────────────────┴─────────────┴───────────┘

-- For ReplacingMergeTree -> https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree

OPTIMIZE TABLE test_delete FINAL;

Ok.

0 rows in set. Elapsed: 2.230 sec. -- 40 times slower!!!

SELECT *
FROM test_delete
WHERE key = 400000

┌────key─┬─ts─┬─value_a──────────────────┬─value_b──────────────────────────┬─value_c─────┬─is_active─┐
│ 400000 │  2 │ totally different string │ another totally different string │ last string │         1 │
└────────┴────┴──────────────────────────┴──────────────────────────────────┴─────────────┴───────────┘
```

Soft Deletion (via ALTER UPDATE): A quicker approach that does not involve physical data deletion but rather updates the tombstone column.
Hard Deletion (via ALTER DELETE): Can take significantly longer, especially with large datasets stored in remote storage like S3.

6. Optimizing for Faster Deletion with S3 Storage
If using S3 for storage, the DELETE operation becomes even slower due to the overhead of handling remote data. Here’s an example with a table using S3-backed storage:

```sql
CREATE TABLE test_delete
(
    `key` UInt32,
    `value_a` String,
    `value_b` String,
    `value_c` String,
    `is_deleted` UInt8 DEFAULT 0
)
ENGINE = MergeTree
ORDER BY key
SETTINGS storage_policy = 's3tiered';

INSERT INTO test_delete (key, value_a, value_b, value_c) SELECT
    number,
    concat('some_looong_string', toString(number)),
    concat('another_long_str', toString(number)),
    concat('really long string', toString(arrayMap(i -> cityHash64(i*number), range(50))))
FROM numbers(10000000);

OPTIMIZE TABLE test_delete FINAL;

ALTER TABLE test_delete MOVE PARTITION tuple() TO DISK 's3disk';

SELECT count() FROM test_delete;
┌──count()─┐
│ 10000000 │
└──────────┘
1 row in set. Elapsed: 0.002 sec.
```

7. DELETE Using ALTER UPDATE and Row Policy
You can also control visibility at the query level using row policies. For example, to only show rows where is_active = 1:

To delete a row using ALTER UPDATE:

```sql
CREATE ROW POLICY pol1 ON test_delete USING is_active=1 TO all;

SELECT count() FROM test_delete;  -- select count() became much slower, it reads data now, not metadata
┌──count()─┐
│ 10000000 │
└──────────┘
1 row in set. Elapsed: 0.314 sec. Processed 10.00 million rows, 10.00 MB (31.84 million rows/s., 31.84 MB/s.)

ALTER TABLE test_delete UPDATE is_active = 0 WHERE (key = 400000) settings mutations_sync = 2;
0 rows in set. Elapsed: 1.256 sec.

SELECT count() FROM test_delete;
┌─count()─┐
│ 9999999 │
└─────────┘
```
This impacts the performance of queries like SELECT count(), as ClickHouse now needs to scan data instead of reading metadata.

8. DELETE Using ALTER DELETE - https://clickhouse.com/docs/en/sql-reference/statements/alter/delete
To delete a row using ALTER DELETE:

```sql
ALTER TABLE test_delete DELETE WHERE (key = 400001) settings mutations_sync = 2;
0 rows in set. Elapsed: 955.672 sec.

SELECT count() FROM test_delete;
┌─count()─┐
│ 9999998 │
└─────────┘
```
This operation may take significantly longer compared to soft deletions (around 955 seconds in this example for large datasets):

9. DELETE Using DELETE Statement - https://clickhouse.com/docs/en/sql-reference/statements/delete
The DELETE statement can also be used to remove data from a table:

```sql
DELETE FROM test_delete WHERE (key = 400002);
0 rows in set. Elapsed: 1.281 sec.

SELECT count() FROM test_delete;
┌─count()─┐
│ 9999997 │
└─────────┘
```
This operation is faster, with an elapsed time of around 1.28 seconds in this case:

The choice between ALTER UPDATE and ALTER DELETE depends on your use case. For soft deletes, updating a tombstone column is significantly faster and easier to manage. However, if you need to physically remove rows, be mindful of the performance costs, especially with remote storage like S3.
