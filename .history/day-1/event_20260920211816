# Exploring the `event` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from event"
```

Output:

```
alert_event
alert_event_local
alert_record
alert_record_local
event
event_local
event_metrics.1s
event_metrics.1s_agg
event_metrics.1s_local
event_metrics.1s_mv
file_event
file_event_local
file_event_metrics.1s
file_event_metrics.1s_agg
file_event_metrics.1s_local
file_event_metrics.1s_mv
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='event'"
```

Output:

```
alert_event                   Distributed
alert_event_local             ReplacingMergeTree
alert_record                  Distributed
alert_record_local            MergeTree
event                         Distributed
event_local                   MergeTree
event_metrics.1s              Distributed
event_metrics.1s_agg          AggregatingMergeTree
event_metrics.1s_local        View
event_metrics.1s_mv           MaterializedView
file_event                    Distributed
file_event_local              MergeTree
file_event_metrics.1s         Distributed
file_event_metrics.1s_agg     AggregatingMergeTree
file_event_metrics.1s_local   View
file_event_metrics.1s_mv      MaterializedView
```

| Group | Table you query | Internal tables | What does it store? |
|---|---|---|---|
| Alerts (events) | `alert_event` | `alert_event_local` | Alert events (alert state changes) |
| Alerts (records) | `alert_record` | `alert_record_local` | Alert records (the full record of each alert) |
| Resource events | `event` | `event_local` | Infrastructure events such as creating/deleting/restarting a Pod |
| File events | `file_event` | `file_event_local` | File operations (read/write) captured by the Agent |

So in practice you have 4 main tables that you query: `alert_event`, `alert_record`, `event`, `file_event`.

### New types in the engine column

| Type | Meaning |
|---|---|
| `ReplacingMergeTree` | Like MergeTree, but it replaces duplicate rows (with the same key) with the latest one. Suitable for `alert_event` because the alert state gets updated |
| `AggregatingMergeTree` | Stores pre-aggregated results (like a sum or an average per second) |
| `View` | A saved query; it does not store data itself |
| `MaterializedView` | Works automatically: whenever new data comes in, it computes the aggregation and writes it into another table |

---

## Metrics tables (`event_metrics` and `file_event_metrics`)

Note that their names contain a dot: `event_metrics.1s`. This is in fact a single database named `event_metrics`... no, careful: the command you ran shows only the tables of the `event` database, so the dot here is part of the table name (its name is `event_metrics.1s`). Each one works with this chain:

```
event_local / file_event_local  (raw data)
        │
        ▼  (MaterializedView: runs automatically on every insert)
event_metrics.1s_mv
        │
        ▼  (writes into)
event_metrics.1s_agg  (AggregatingMergeTree: aggregated metrics per second)
        │
        ▼  (View: reads and un-aggregates)
event_metrics.1s_local
        │
        ▼
event_metrics.1s  (Distributed: the one you query)
```

The same chain applies to `file_event_metrics.1s`.

| Table | Role |
|---|---|
| `event_metrics.1s` / `file_event_metrics.1s` | You query it: metrics aggregated every second (like the number of events) |
| `..._agg` | The actual storage of the aggregation |
| `..._local` (View) | Intermediate layer |
| `..._mv` | The automatic aggregation mechanism |

---

# 1) `alert_event`

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe event.alert_event"
```

Columns of `event.alert_event`

This table has no comments from DeepFlow (except `time`), so the following explanation is based on the column names and what I know about DeepFlow, not on an official text.

| Column | Type | Explanation |
|---|---|---|
| `time` | DateTime | Event time in seconds (the number is the schema version) |
| `_id` | UInt64 | Unique record ID |
| `policy_id` | UInt32 | ID of the alert policy (Policy) that triggered the event |
| `policy_type` | UInt8 | Policy type (numeric) |
| `alert_policy` | String | Name of the alert policy |
| `metric_value` | Float64 | The metric value at the time of the alert (number) |
| `metric_value_str` | String | The metric value as text |
| `event_level` | UInt8 | Event severity level (numeric) |
| `target_tags` | String | Tags of the target the alert happened on (like a Pod or a service) |
| `tag_string_names` | Array(String) | Names of the string tags |
| `tag_string_values` | Array(String) | Values of the string tags (correspond to the names in order) |
| `tag_int_names` | Array(String) | Names of the numeric tags |
| `tag_int_values` | Array(Int64) | Values of the numeric tags (correspond to the names in order) |
| `trigger_threshold` | String | The trigger threshold the metric exceeded |
| `metric_unit` | String | Unit of the metric (like ms or %) |
| `custom_tag_names` | Array(String) | Names of custom tags |
| `custom_tag_values` | Array(String) | Values of the custom tags |
| `_target_uid` | String | Unique ID of the target |
| `_query_region` | String | The region where the alert query was executed |
| `team_id` | UInt16 | Team ID |
| `user_id` | UInt32 | User ID |
| `event_id` | String | Event ID (links the events related to the same alert) |
| `start_time` | DateTime | Alert start time |
| `end_time` | DateTime | Alert end time |
| `duration` | UInt32 | Alert duration (probably in seconds) |
| `state` | UInt32 | Alert state (numeric, like active or resolved) |
| `alert_time` | UInt64 | Alert time (a large number, probably a timestamp) |

---

# 2) `alert_record`

Columns of `event.alert_record`

This table is almost like `alert_event`, but shorter: it has only the first 21 columns, and it does not contain the last six columns (`start_time`, `end_time`, `duration`, `state`, `alert_time`).

| Column | Type | Explanation |
|---|---|---|
| `time` | DateTime | Record time in seconds (the number is the schema version) |
| `_id` | UInt64 | Unique record ID |
| `policy_id` | UInt32 | ID of the alert policy |
| `policy_type` | UInt8 | Policy type (numeric) |
| `alert_policy` | String | Name of the alert policy |
| `metric_value` | Float64 | The metric value at the time of the alert |
| `metric_value_str` | String | The metric value as text |
| `event_level` | UInt8 | Event severity level (numeric) |
| `target_tags` | String | Tags of the target the alert happened on |
| `tag_string_names` | Array(String) | Names of the string tags |
| `tag_string_values` | Array(String) | Values of the string tags (correspond to the names in order) |
| `tag_int_names` | Array(String) | Names of the numeric tags |
| `tag_int_values` | Array(Int64) | Values of the numeric tags (correspond to the names in order) |
| `trigger_threshold` | String | Trigger threshold |
| `metric_unit` | String | Unit of the metric |
| `custom_tag_names` | Array(String) | Names of custom tags |
| `custom_tag_values` | Array(String) | Values of the custom tags |
| `_target_uid` | String | Unique ID of the target |
| `_query_region` | String | The region where the query was executed |
| `team_id` | UInt16 | Team ID |
| `user_id` | UInt32 | User ID |
| `event_id` | String | Event ID (links the records of the same alert) |

---

# 3) `event`

Columns of `event.event`

The comments here are in Chinese, so I translated them for you in the third column.

| Column | Type | Description (from DeepFlow) | Explanation |
|---|---|---|---|
| `time` | DateTime | v7.1.7.3 | Event time in seconds (the number is the schema version) |
| `_id` | UInt64 | (no description) | Unique record ID |
| `start_time` | DateTime64(6) | 精度: 微秒 = "precision: microseconds" | Event start time with microsecond precision |
| `end_time` | DateTime64(6) | 精度: 微秒 | Event end time with microsecond precision |
| `tagged` | UInt8 | 标签是否为填充, 用于调试 = "whether the tags are filled, for debugging" | Debugging field: whether the tags were added to the event |
| `signal_source` | UInt8 | 事件来源 = "event source" | Where the event came from (numeric) |
| `event_type` | String | 事件类型 = "event type" | Event type |
| `event_desc` | String | 事件信息 = "event information" | Event description (the most important column for reading) |
| `process_kname` | String | 系统进程 = "system process" | Name of the system process related to the event |
| `gprocess_id` | UInt32 | 全局进程ID = "global process ID" | Global process ID |
| `region_id` | UInt16 | 云平台区域ID = "cloud platform region ID" | Region ID |
| `az_id` | UInt16 | 可用区ID = "availability zone ID" | Availability zone ID |
| `l3_epc_id` | Int32 | ip对应的EPC ID = "EPC ID matching the IP" | Virtual network (VPC) ID |
| `host_id` | UInt16 | 宿主机ID = "host server ID" | Host server ID |
| `pod_id` | UInt32 | 容器ID = "container ID" | Pod ID |
| `pod_node_id` | UInt32 | 容器节点ID = "container node ID" | Kubernetes node ID |
| `pod_ns_id` | UInt16 | 容器命名空间ID = "Namespace ID" | Namespace ID |
| `pod_cluster_id` | UInt16 | 容器集群ID = "container cluster ID" | Cluster ID |
| `pod_group_id` | UInt32 | 容器组ID = "container group ID" | Workload ID (e.g. Deployment) |
| `l3_device_type` | UInt8 | 资源类型 = "resource type" | Resource type |
| `l3_device_id` | UInt32 | 资源ID = "resource ID" | Resource ID |
| `service_id` | UInt32 | 服务ID = "service ID" | Service ID |
| `agent_id` | UInt16 | 采集器ID = "collector ID" | DeepFlow Agent ID |
| `subnet_id` | UInt16 | (no description) | Subnet ID |
| `is_ipv4` | UInt8 | (no description) | Whether the address is IPv4 (1) or IPv6 (0) |
| `ip4` | IPv4 | (no description) | IPv4 address |
| `ip6` | IPv6 | (no description) | IPv6 address |
| `team_id` | UInt16 | Team ID | Team ID |
| `auto_instance_id` | UInt32 | (no description) | Instance ID (automatically prefers the Pod) |
| `auto_instance_type` | UInt8 | (no description) | Instance type |
| `auto_service_id` | UInt32 | (no description) | Service ID (automatically prefers the Service) |
| `auto_service_type` | UInt8 | (no description) | Service type |
| `app_instance` | String | app instance | Application instance name |
| `attribute_names` | Array(String) | 额外的属性 = "extra attributes" | Names of the extra attributes |
| `attribute_values` | Array(String) | 额外的属性对应的值 = "values of the extra attributes" | Attribute values (correspond to the names in order) |

---

# 4) `file_event`

The table has four columns: name, type, original description (translated from Chinese), and the Arabic explanation. The columns with no official comment were explained from their names and what I know about DeepFlow.

### 1) Time

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 1 | `time` | DateTime | v7.1.7.3 | Event time in seconds (the number is the schema version) |
| 2 | `_id` | UInt64 | - | Unique record ID |
| 3 | `start_time` | DateTime64(6) | 精度: 微秒 = precision: microseconds | Operation start time |
| 4 | `end_time` | DateTime64(6) | 精度: 微秒 | Operation end time |

### 2) Event information

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 5 | `tagged` | UInt8 | 标签是否为填充, 用于调试 = whether the tags are filled (for debugging) | Debugging field: whether the tags were added to the event |
| 6 | `signal_source` | UInt8 | 事件来源 = event source | Event source (numeric) |
| 7 | `event_type` | String | 事件类型 = event type | Event type (like read or write) |
| 8 | `event_desc` | String | 事件信息 = event information | Event description |
| 9 | `process_kname` | String | 系统进程 = system process | Name of the process that performed the operation |
| 10 | `gprocess_id` | UInt32 | 全局进程ID = global process ID | Global process ID |

### 3) Infrastructure resources (numeric IDs)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 11 | `region_id` | UInt16 | 云平台区域ID = platform region ID | Region ID |
| 12 | `az_id` | UInt16 | 可用区ID = availability zone ID | Availability zone ID |
| 13 | `l3_epc_id` | Int32 | ip对应的EPC ID = EPC ID matching the IP | Virtual network (VPC) ID |
| 14 | `host_id` | UInt16 | 宿主机ID = host server ID | Host server ID |
| 15 | `pod_id` | UInt32 | 容器ID = container ID | Pod ID |
| 16 | `pod_node_id` | UInt32 | 容器节点ID = container node ID | Kubernetes node ID |
| 17 | `pod_ns_id` | UInt16 | 容器命名空间ID = Namespace ID | Namespace ID |
| 18 | `pod_cluster_id` | UInt16 | 容器集群ID = container cluster ID | Cluster ID |
| 19 | `pod_group_id` | UInt32 | 容器组ID = container group ID | Workload ID (e.g. Deployment) |
| 20 | `l3_device_type` | UInt8 | 资源类型 = resource type | Resource type |
| 21 | `l3_device_id` | UInt32 | 资源ID = resource ID | Resource ID |
| 22 | `service_id` | UInt32 | 服务ID = service ID | Service ID |
| 23 | `agent_id` | UInt16 | 采集器ID = collector ID | DeepFlow Agent ID |

### 4) Network and identity

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 24 | `subnet_id` | UInt16 | - | Subnet ID |
| 25 | `is_ipv4` | UInt8 | - | Whether the address is IPv4 (1) or IPv6 (0) |
| 26 | `ip4` | IPv4 | - | IPv4 address |
| 27 | `ip6` | IPv6 | - | IPv6 address |
| 28 | `team_id` | UInt16 | Team ID | Team ID |
| 29 | `auto_instance_id` | UInt32 | - | Instance ID (automatically prefers the Pod) |
| 30 | `auto_instance_type` | UInt8 | - | Instance type |
| 31 | `auto_service_id` | UInt32 | - | Service ID (automatically prefers the Service) |
| 32 | `auto_service_type` | UInt8 | - | Service type |
| 33 | `app_instance` | String | app instance | Application instance name |

### 5) Extra attributes (Key-Value)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 34 | `attribute_names` | Array(String) | 额外的属性 = extra attributes | Names of the extra attributes |
| 35 | `attribute_values` | Array(String) | 额外的属性对应的值 = values of the extra attributes | Attribute values (correspond to the names in order) |

### 6) File columns (specific to this table)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 36 | `bytes` | UInt32 | - | Number of bytes read or written |
| 37 | `duration` | UInt64 | 精度: 微秒 = precision: microseconds | Duration of the operation in microseconds |
| 38 | `file_name` | String | 文件名 = file name | File name |
| 39 | `file_type` | UInt8 | 文件类型 = file type | File type (numeric) |
| 40 | `offset` | UInt64 | 读写偏移 = read/write offset | Read/write position inside the file |
| 41 | `syscall_thread` | UInt32 | - | ID of the thread that executed the syscall |
| 42 | `syscall_coroutine` | UInt32 | - | Coroutine ID (e.g. Go) |
| 43 | `mount_source` | String | - | Source of the mount point (device/file system) |
| 44 | `mount_point` | String | - | Mount point |
| 45 | `file_dir` | String | - | The folder where the file is located |