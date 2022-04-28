# Timestamp Index

Timestamp stores value as millisecond epoch long value.&#x20;

Typical workload for timestamp is filtering on a time range and group by with different time granularities(days/month/etc).

The current implementation requires the query executor to extract values, apply the function then do filter/groupBy, no leverage on dictionary or index.

## Supported data type

So far, Timestamp index only supports TIMESTAMP data type.

## Timestamp Index

Users can configure the most useful granularities for a Timestamp data type column.

1. Pinot will pre-generate one column per time granularity with forward index and range index. The naming convention is `$${ts_column_name}$${ts_granularity}`, e.g. Timestamp column `ts` with granularities `DAY`, `MONTH` will have two extra columns generated: `$ts$DAY` and `$ts$MONTH`.
2. Query overwrite for predicate and selection/group by:\
   2.1 GROUP BY: functions like `dateTrunc('DAY', ts)` will be translated to use the underly column `$ts$DAY` to fetch data.\
   2.2 PREDICATE: range index is auto-built for all granularity columns.

Example query usage:

```
select count(*), datetrunc('WEEK', ts) as tsWeek from airlineStats WHERE tsWeek > fromDateTime('2014-01-16', 'yyyy-MM-dd') group by tsWeek  limit 10
```

Some preliminary benchmark shows the query perf over 2.7 billion records improved from 45 secs to 4.2 secs

```
select dateTrunc('YEAR', event_time) as y, dateTrunc('MONTH', event_time) as m,  sum(pull_request_commits) from githubEvents group by y, m limit 1000  Option(timeoutMs=3000000)
```



![Without Timestamp Index](https://user-images.githubusercontent.com/1202120/160910329-0d9ca637-dc95-4137-8c79-2f66cc8fbabf.png)

vs.

![With Timestamp Index](https://user-images.githubusercontent.com/1202120/160910364-48424875-1967-42d3-9a76-bdf1ee81a4ca.png)

### Usage

Timestamp index is configured per column basis inside the fieldConfigList section in table config.

Users need to specify `TIMESTAMP` as part of the `indexTypes`. Then in the field timestampConfig, specify the granularities that you want to index.

Sample config:

```
{
  "tableName": "airlineStats",
  "tableType": "OFFLINE",
  "segmentsConfig": {
    "timeColumnName": "DaysSinceEpoch",
    "timeType": "DAYS",
    "segmentPushType": "APPEND",
    "segmentAssignmentStrategy": "BalanceNumSegmentAssignmentStrategy",
    "replication": "1"
  },
  "tenants": {},
  "fieldConfigList": [
    {
      "name": "ts",
      "encodingType": "DICTIONARY",
      "indexTypes": ["TIMESTAMP"],
      "timestampConfig": {
        "granularities": [
          "DAY",
          "WEEK",
          "MONTH"
        ]
      }
    }
  ],
  "tableIndexConfig": {
    "loadMode": "MMAP"
  },
  "metadata": {
    "customConfigs": {}
  },
  "ingestionConfig": {}
}

```
