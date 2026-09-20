# Exploring the `prometheus` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from prometheus"
```

Output:

```
samples
samples_local
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='prometheus'"
```

Output:

```
samples         Distributed
samples_local   MergeTree
```

---

## Columns of `prometheus.samples` (all 34 columns)

The output arrived in full (the last column is `agent_id`). This table combines two ideas seen earlier: resource columns (like `ext_metrics.metrics`) + Prometheus-specific columns. The Chinese comments are translated.

### 1) Time and value

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 1 | `time` | DateTime | v7.1.7.3 | Sample time in seconds (the number is the schema version) |
| 2 | `value` | Float64 | - | The metric value itself (the number) |

### 2) Metric identity (Prometheus-specific)

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 3 | `metric_id` | UInt32 | encoded ID of the metric name | Encoded ID of the metric name (e.g. `node_cpu_seconds_total`) |
| 4 | `target_id` | UInt32 | the encoded ID of the target | Encoded ID of the target the metric was collected from |
| 5 | `team_id` | UInt16 | the team ID | Team ID |
| 6-13 | `app_label_value_id_1` ... `app_label_value_id_8` | UInt32 | - | Encoded IDs of label values, up to 8 labels |

**The idea:** instead of storing the metric name and labels as text (which uses more space), DeepFlow stores them as numbers, and they are translated back to text through tables in the `flow_tag` database. This is an inference from the names and what I know about DeepFlow, and it will be verified when inspecting `flow_tag`.

### 3) IP address

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 14 | `ip4` | IPv4 | IPv4地址 | IPv4 address of the metric source |
| 15 | `ip6` | IPv6 | IPV6地址 | IPv6 address of the metric source |
| 16 | `is_ipv4` | UInt8 | 是否IPV4地址. 0: ip6 valid, 1: ip4 valid | Whether the address is IPv4 (1) or IPv6 (0) |

### 4) Infrastructure resources

| # | Column | Type | Explanation |
|---|---|---|---|
| 17 | `region_id` | UInt16 | Region ID |
| 18 | `az_id` | UInt16 | Availability zone ID |
| 19 | `host_id` | UInt16 | Host server ID |
| 20 | `l3_epc_id` | Int32 | Virtual network (VPC) ID |
| 21 | `l3_device_type` | UInt8 | Type of the resource matching the IP |
| 22 | `l3_device_id` | UInt32 | ID of the resource matching the IP |
| 23 | `subnet_id` | UInt16 | Subnet ID (0 = none) |
| 24 | `gprocess_id` | UInt32 | Global process ID |
| 25 | `agent_id` | UInt16 | DeepFlow Agent ID |

### 5) Kubernetes resources

| # | Column | Type | Explanation |
|---|---|---|---|
| 26 | `pod_cluster_id` | UInt16 | Cluster ID |
| 27 | `pod_node_id` | UInt32 | Kubernetes node ID |
| 28 | `pod_ns_id` | UInt16 | Namespace ID |
| 29 | `pod_group_id` | UInt32 | Workload ID (e.g. Deployment) |
| 30 | `pod_id` | UInt32 | Pod ID |
| 31 | `service_id` | UInt32 | Service ID |

### 6) Automatic (Auto) fields

| # | Column | Type | Original description (translated) | Explanation |
|---|---|---|---|---|
| 32 | `auto_instance_id` | UInt32 | Priority: `pod_id` ← `pod_node_id` ← `l3_device_id` | Instance ID; DeepFlow picks the first available value in this order |
| 33 | `auto_instance_type` | UInt8 | 0 = IP with no resource, 0-100 = deviceType (10 = pod, 14 = podNode), 101-200 = abstract resources (101 = podGroup, 102 = service), 201-255 = other | Instance type |
| 34 | `auto_service_id` | UInt32 | Priority: `service_id` ← `pod_node_id` ← `l3_device_id` | Service ID; the priority order is different |
| 35 | `auto_service_type` | UInt8 | Same type codes as above | Service type |

**Note:** after counting precisely, the output has 35 columns, not 34 as written in the heading (I miscounted).

---

## Difference between `ext_metrics.metrics` and `prometheus.samples`

| | `ext_metrics.metrics` | `prometheus.samples` |
|---|---|---|
| Metric name | Text in an array (`metrics_float_names`) | Encoded number (`metric_id`) |
| Labels | Text in arrays | Encoded numbers (`app_label_value_id_1..8`) |
| Value | Array `metrics_float_values` | A single `value` column |
| Resource columns | Present | Present (almost the same) |

In other words, `prometheus.samples` is more compact but harder to read directly, because you need to translate the IDs through `flow_tag`.