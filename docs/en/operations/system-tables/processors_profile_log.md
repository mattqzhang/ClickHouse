# system.processors_profile_log {#system-processors_profile_log}

This table contains profiling on processors level (that you can find in [`EXPLAIN PIPELINE`](../../sql-reference/statements/explain.md#explain-pipeline)).

Columns:

-   `event_date` ([Date](../../sql-reference/data-types/date.md)) — The date when the event happened.
-   `event_time` ([DateTime64](../../sql-reference/data-types/datetime64.md)) — The date and time when the event happened.
-   `id` ([UInt64](../../sql-reference/data-types/int-uint.md)) — ID of processor
-   `parent_ids` ([Array(UInt64)](../../sql-reference/data-types/array.md)) — Parent processors IDs
-   `query_id` ([String](../../sql-reference/data-types/string.md)) — ID of the query
-   `name` ([LowCardinality(String)](../../sql-reference/data-types/lowcardinality.md)) — Name of the processor.
-   `elapsed_us` ([UInt64](../../sql-reference/data-types/int-uint.md)) — Number of microseconds this processor was executed.
-   `input_wait_elapsed_us` ([UInt64](../../sql-reference/data-types/int-uint.md)) — Number of microseconds this processor was waiting for data (from other processor).
-   `output_wait_elapsed_us` ([UInt64](../../sql-reference/data-types/int-uint.md)) — Number of microseconds this processor was waiting because output port was full.

**Example**

Query:

``` sql
EXPLAIN PIPELINE
SELECT sleep(1)

┌─explain─────────────────────────┐
│ (Expression)                    │
│ ExpressionTransform             │
│   (SettingQuotaAndLimits)       │
│     (ReadFromStorage)           │
│     SourceFromSingleChunk 0 → 1 │
└─────────────────────────────────┘

SELECT sleep(1)
SETTINGS log_processors_profiles = 1

Query id: feb5ed16-1c24-4227-aa54-78c02b3b27d4

┌─sleep(1)─┐
│        0 │
└──────────┘

1 rows in set. Elapsed: 1.018 sec.

SELECT
    name,
    elapsed_us,
    input_wait_elapsed_us,
    output_wait_elapsed_us
FROM system.processors_profile_log
WHERE query_id = 'feb5ed16-1c24-4227-aa54-78c02b3b27d4'
ORDER BY name ASC
```

Result:

``` text
┌─name────────────────────┬─elapsed_us─┬─input_wait_elapsed_us─┬─output_wait_elapsed_us─┐
│ ExpressionTransform     │    1000497 │                  2823 │                    197 │
│ LazyOutputFormat        │         36 │               1002188 │                      0 │
│ LimitsCheckingTransform │         10 │               1002994 │                    106 │
│ NullSource              │          5 │               1002074 │                      0 │
│ NullSource              │          1 │               1002084 │                      0 │
│ SourceFromSingleChunk   │         45 │                  4736 │                1000819 │
└─────────────────────────┴────────────┴───────────────────────┴────────────────────────┘
```

Here you can see:

-  `ExpressionTransform` was executing `sleep(1)` function, so it `work` will takes 1e6, and so `elapsed_us` > 1e6.
-  `SourceFromSingleChunk` need to wait, because `ExpressionTransform` does not accept any data during execution of `sleep(1)`, so it will be in `PortFull` state for 1e6 us, and so `output_wait_elapsed_us` > 1e6.
-  `LimitsCheckingTransform`/`NullSource`/`LazyOutputFormat` need to wait until `ExpressionTransform` will execute `sleep(1)` to process the result, so `input_wait_elapsed_us` > 1e6.

**See Also**

-   [`EXPLAIN PIPELINE`](../../sql-reference/statements/explain.md#explain-pipeline)