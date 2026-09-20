# Exploring the `deepflow_tenant` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from deepflow_tenant"
```

Output:

```
deepflow_collector
deepflow_collector_local
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='deepflow_tenant'"
```

Output:

```
deepflow_collector         Distributed
deepflow_collector_local   MergeTree
```

---

## Columns of `deepflow_tenant.deepflow_collector`

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe deepflow_tenant.deepflow_collector"
```

The structure is exactly identical to the `deepflow_admin.deepflow_server` table, and the only difference is that this time the output arrived in full (the description of the last column appeared).

| Column | Type | Description (from DeepFlow) | Explanation |
|---|---|---|---|
| `time` | DateTime | v7.1.7.3 | Time the data was recorded, in seconds (the number is the schema version) |
| `virtual_table_name` | String | 虚拟表名 = "virtual table name" | Name of the module or category the metrics came from |
| `team_id` | UInt16 | 团队ID = "team ID" | Team ID (Team) |
| `tag_names` | Array(String) | 额外的tag = "extra tags" | Names of the extra tags |
| `tag_values` | Array(String) | 额外的tag对应的值 = "values of the extra tags" | Tag values, correspond to the names in order |
| `metrics_float_names` | Array(String) | 额外的float类型metrics = "extra float-type metrics" | Names of the numeric metrics |
| `metrics_float_values` | Array(Float64) | 额外的float metrics值 = "values of the extra float metrics" | Metric values, correspond to the names in order |