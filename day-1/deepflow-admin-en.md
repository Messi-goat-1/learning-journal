# Exploring the `deepflow_admin` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from deepflow_admin"
```

Output:

```
Defaulted container "clickhouse" out of: clickhouse, clickhouse-init (init)
deepflow_server
deepflow_server_local
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='deepflow_admin'"
```

Output:

```
Defaulted container "clickhouse" out of: clickhouse, clickhouse-init (init)
deepflow_server         Distributed
deepflow_server_local   MergeTree
```

---

## Columns of `deepflow_admin.deepflow_server`

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe deepflow_admin.deepflow_server"
```

The comments in this table are written in Chinese, so I translated them for you in the second column:

| Column | Type | Description (from DeepFlow) | Explanation |
|---|---|---|---|
| `time` | DateTime | v7.1.7.3 | Time the data was recorded, in seconds (the number is the schema version) |
| `virtual_table_name` | String | 虚拟表名 = "virtual table name" | Name of the category or virtual table the metrics belong to (for example a specific module inside the Server) |
| `team_id` | UInt16 | 团队ID = "team ID" | Team ID (Team) |
| `tag_names` | Array(String) | 额外的tag = "extra tags" | Names of the extra tags |
| `tag_values` | Array(String) | 额外的tag对应的值 = "values of the extra tags" | Tag values, correspond to the names in order |
| `metrics_float_names` | Array(String) | 额外的float类型metrics = "extra float-type metrics" | Names of the numeric metrics |
| `metrics_float_values` | Array(Float64) | (no description) | Metric values, correspond to the names in order |