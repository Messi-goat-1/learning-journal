# Exploring the `ext_metrics` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from ext_metrics"
```

Output:

```
metrics
metrics_local
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='ext_metrics'"
```

| Table | Type | Role |
|---|---|---|
| `metrics_local` | MergeTree | The real table where the data is stored |
| `metrics` | Distributed | An interface on top of it, and it is the one you query |

---

## Columns of `ext_metrics.metrics` (all 30 columns)

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe ext_metrics.metrics"
```

This table combines two ideas seen earlier: resource columns (like `event.event`) + Key-Value arrays (like `deepflow_server`). The comments are Chinese and translated.

### 1) Time

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 1 | `time` | DateTime | v7.1.7.3 | Time the metric was recorded, in seconds (the number is the schema version) |

### 2) IP address

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 2 | `ip4` | IPv4 | IPv4地址 = IPv4 address | IPv4 address of the metric source |
| 3 | `ip6` | IPv6 | IPV6地址 = IPv6 address | IPv6 address of the metric source |
| 4 | `is_ipv4` | UInt8 | 是否IPV4地址. 0: 否, ip6 valid, 1: 是, ip4 valid | Whether the address is IPv4 (1) or IPv6 (0) |

### 3) Infrastructure resources (numeric IDs)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 5 | `region_id` | UInt16 | ip对应的云平台区域ID = region ID matching the IP | Region ID |
| 6 | `az_id` | UInt16 | 可用区ID = availability zone ID | Availability zone ID |
| 7 | `host_id` | UInt16 | 宿主机ID = host server ID | Host server ID |
| 8 | `l3_epc_id` | Int32 | ip对应的EPC ID | Virtual network (VPC) ID |
| 9 | `l3_device_type` | UInt8 | ip对应的资源类型 = resource type matching the IP | Resource type |
| 10 | `l3_device_id` | UInt32 | ip对应的资源ID = resource ID matching the IP | Resource ID |
| 11 | `subnet_id` | UInt16 | ip对应的子网ID(0: 未找到) = subnet ID (0: not found) | Subnet ID |
| 12 | `gprocess_id` | UInt32 | 全局进程ID = global process ID | Global process ID |
| 13 | `agent_id` | UInt16 | 采集器的ID = collector ID | DeepFlow Agent ID |

### 4) Kubernetes resources

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 14 | `pod_cluster_id` | UInt16 | 容器集群ID = container cluster ID | Cluster ID |
| 15 | `pod_node_id` | UInt32 | 容器节点ID = container node ID | Kubernetes node ID |
| 16 | `pod_ns_id` | UInt16 | 容器命名空间ID = Namespace ID | Namespace ID |
| 17 | `pod_group_id` | UInt32 | 容器工作负载ID = Workload ID | Workload ID (e.g. Deployment) |
| 18 | `pod_id` | UInt32 | 容器POD ID = Pod ID | Pod ID |
| 19 | `service_id` | UInt32 | ip对应的服务ID = service ID | Service ID |

### 5) Automatic (Auto) fields

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 20 | `auto_instance_id` | UInt32 | Preferred resource: the Pod first. Priority: `pod_id` ← `pod_node_id` ← `l3_device_id` | Instance ID; DeepFlow picks the first available value in this order |
| 21 | `auto_instance_type` | UInt8 | Resource type: 0 = IP with no resource, 0-100 = deviceType (10 = pod, 14 = podNode), 101-200 = abstract resources (101 = podGroup, 102 = service), 201-255 = other | Instance type, and this table explains the value codes |
| 22 | `auto_service_id` | UInt32 | Preferred resource: the Service first. Priority: `service_id` ← `pod_node_id` ← `l3_device_id` | Service ID; the priority order is different from `auto_instance_id` |
| 23 | `auto_service_type` | UInt8 | Same type codes as above | Service type |

### 6) Metrics and tags (Key-Value)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 24 | `virtual_table_name` | String | 虚拟表名 = virtual table name | Name of the source or category the metrics came from |
| 25 | `team_id` | UInt16 | 团队ID = team ID | Team ID |
| 26 | `tag_names` | Array(String) | 额外的tag = extra tags | Tag names |
| 27 | `tag_values` | Array(String) | 额外的tag对应的值 = tag values | Tag values (correspond to the names in order) |
| 28 | `metrics_float_names` | Array(String) | 额外的float类型metrics = extra float-type metrics | Names of the numeric metrics |
| 29 | `metrics_float_values` | Array(Float64) | 额外的float metrics值 = metric values | Metric values (correspond to the names in order) |