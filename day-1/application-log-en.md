# Exploring the `application_log` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from application_log"
```

It showed me:

```
log
log_local
```

---

## Table engines

I ran a command to confirm:

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='application_log'"
```

```
log         Distributed
log_local   MergeTree
```

---

## Columns of `application_log.log`

I ran this to find out the columns:

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe application_log.log"
```

| Column | Type | Description (from DeepFlow) | Explanation |
|---|---|---|---|
| `time` | DateTime | v7.1.7.3 | Record time in seconds (the number is the schema version) |
| `timestamp` | DateTime64(6) | precision: us | Record time with microsecond precision |
| `_id` | UInt64 | Unique ID | Unique record ID |
| `_type` | Enum8 | log type | Record type: user=1, system=2, audit=3, agent=4 |
| `trace_id` | String | Trace ID | Trace ID to link the record to a specific request |
| `span_id` | String | Span ID | ID of the span inside the trace |
| `trace_flags` | UInt32 | W3C trace flag | Trace flags (not currently supported) |
| `severity_number` | UInt8 | log level id | Severity level as a number |
| `body` | String | log content | The record text itself |
| `app_service` | LowCardinality(String) | Application Service | Name of the service that produced the record |
| `gprocess_id` | UInt32 | Global Process ID | Global process ID |
| `agent_id` | UInt16 | Agent ID | DeepFlow Agent ID |
| `region_id` | UInt16 | Region ID | Region ID |
| `az_id` | UInt16 | Availability Zone ID | Availability zone ID |
| `l3_epc_id` | Int32 | VPC ID | Virtual network (VPC) ID |
| `host_id` | UInt16 | Hypervisor ID | Host server ID (Hypervisor) |
| `pod_id` | UInt32 | K8s POD ID | Pod ID |
| `pod_node_id` | UInt32 | K8s Node ID | Kubernetes node ID |
| `pod_ns_id` | UInt16 | K8s Namespace ID | Namespace ID |
| `pod_cluster_id` | UInt16 | K8s Cluster ID | Cluster ID |
| `pod_group_id` | UInt32 | K8s Workload ID | Workload ID (e.g. Deployment) |
| `l3_device_type` | UInt8 | Resource Type | Resource type |
| `l3_device_id` | UInt32 | Resource ID | Resource ID |
| `service_id` | UInt32 | Service ID | Service ID |
| `subnet_id` | UInt16 | Subnet ID | Subnet ID |
| `is_ipv4` | UInt8 | (no description) | Whether the address is IPv4 (1) or IPv6 (0) |
| `ip4` | IPv4 | (no description) | IPv4 address |
| `ip6` | IPv6 | (no description) | IPv6 address |
| `team_id` | UInt16 | Team ID | Team ID |
| `user_id` | UInt32 | User ID | User ID |
| `auto_instance_id` | UInt32 | Instance - K8s POD First | Instance ID (automatically prefers the Pod) |
| `auto_instance_type` | UInt8 | Type - K8s POD First | Instance type (prefers the Pod) |
| `auto_service_id` | UInt32 | Instance - K8s Service First | Service ID (automatically prefers the Service) |
| `auto_service_type` | UInt8 | Type - K8s Service First | Service type (prefers the Service) |
| `attribute_names` | Array(String) | Extra Attributes | Names of the extra attributes |
| `attribute_values` | Array(String) | value of extra attributes | Values of the extra attributes (correspond to the names in order) |
| `metrics_names` | Array(String) | Extra Metrics | Names of the extra numeric metrics |
| `metrics_values` | Array(Float64) | Extra Metrics values | Values of the extra metrics (correspond to the names in order) |

### Note

I removed the compression columns (DoubleDelta, ZSTD, T64) from the table because they relate to the internal storage method and do not matter when querying.

I use the `application_log.log` table; it is the main one.