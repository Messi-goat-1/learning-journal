# Exploring the `flow_log` Database in DeepFlow

## Listing the tables

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show tables from flow_log"
```

Output:

```
l4_flow_log
l4_flow_log_local
l4_packet
l4_packet_local
l7_flow_log
l7_flow_log_local
l7_packet
l7_packet_local
```

---

## Table engines

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "select name, engine from system.tables where database='flow_log'"
```

Output:

```
l4_flow_log         Distributed
l4_flow_log_local   MergeTree
l4_packet           Distributed
l4_packet_local     MergeTree
l7_flow_log         Distributed
l7_flow_log_local   MergeTree
l7_packet           Distributed
l7_packet_local     MergeTree
```

---

## The four tables you query

| Table | What does it store? | Example use |
|---|---|---|
| `l4_flow_log` | Network flow logs at layer 4 (TCP/UDP): connections, bytes, latency, retransmissions | Who connects to whom? How much data? Are there disconnections? |
| `l7_flow_log` | Request logs at layer 7 (HTTP, DNS, MySQL, Redis...) with tracing data | Which request failed? How long did it take? Which service is slow? |
| `l4_packet` | Raw network packets linked to L4 flows | Deep packet-level analysis (like Wireshark) |
| `l7_packet` | Packets linked to L7 requests | Same idea, but for application requests |

## Difference between `flow_log` and `packet` tables

| | `*_flow_log` | `*_packet` |
|---|---|---|
| Content | Summarized, structured records | Raw packets (can be large) |
| Usage | Primary and most common | Rare, needs to be enabled in the Agent settings |
| Expected state | Usually has data | Usually empty unless packet capture is enabled |

---

# Columns of `flow_log.l7_flow_log` (all columns)

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe flow_log.l7_flow_log"
```

The table is large (121 columns), so I split it into groups. Important: many columns come in two versions, `_0` and `_1`:

- `_0` = the first side: the client, which sent the request
- `_1` = the second side: the server, which responded

So I explain each pair in a single row. The Chinese comments are translated, and the columns with no comment were explained from their names and what I know about DeepFlow.

### 1) Identity and time

| Column | Type | Explanation |
|---|---|---|
| `_id` | UInt64 | Unique record ID |
| `time` | DateTime | Record time in seconds (the number v7.1.7.3 is the schema version) |
| `start_time` | DateTime64(6) | Request start time (precision: microseconds) |
| `end_time` | DateTime64(6) | Request end time (precision: microseconds) |
| `flow_id` | UInt64 | ID of the network flow (L4) the request came from; links this table to `l4_flow_log` |
| `team_id` | UInt16 | Team ID |
| `agent_id` | UInt16 | ID of the Agent that captured the request |

### 2) Resources of both sides (`_0` / `_1` pairs)

All are numeric IDs that are translated into names through the `flow_tag` database:

| Column (without suffix) | Type | Meaning |
|---|---|---|
| `region_id` | UInt16 | Region |
| `az_id` | UInt16 | Availability zone |
| `host_id` | UInt16 | Host server |
| `l3_device_type` | UInt8 | Resource type |
| `l3_device_id` | UInt32 | Resource ID |
| `pod_node_id` | UInt32 | Kubernetes node |
| `pod_ns_id` | UInt16 | Namespace |
| `pod_group_id` | UInt32 | Workload (e.g. Deployment) |
| `pod_id` | UInt32 | Pod |
| `pod_cluster_id` | UInt16 | Cluster |
| `l3_epc_id` | Int32 | Virtual network (VPC) |
| `epc_id` | Int32 | EPC ID (close to the previous one) |
| `subnet_id` | UInt16 | Subnet |
| `service_id` | UInt32 | Service |
| `tag_source` | UInt8 | Source of tag population (numeric) |

Each of these columns has two versions, for example `pod_id_0` (the client's Pod) and `pod_id_1` (the server's Pod).

### 3) Automatic (Auto) fields

| Column | Type | Meaning |
|---|---|---|
| `auto_instance_id_0` / `auto_instance_id_1` | UInt32 | Instance ID for each side (prefers the Pod) |
| `auto_instance_type_0` / `auto_instance_type_1` | UInt8 | Instance type |
| `auto_service_id_0` / `auto_service_id_1` | UInt32 | Service ID for each side (prefers the Service) |
| `auto_service_type_0` / `auto_service_type_1` | UInt8 | Service type |

### 4) IP addresses and ports

| Column | Type | Meaning |
|---|---|---|
| `ip4_0` / `ip4_1` | IPv4 | IPv4 address of the client / server |
| `ip6_0` / `ip6_1` | IPv6 | IPv6 address of the client / server |
| `is_ipv4` | UInt8 | Whether the address is IPv4 (1) or IPv6 (0) |
| `protocol` | UInt8 | Transport protocol (e.g. TCP=6, UDP=17) |
| `client_port` | UInt16 | Client port |
| `server_port` | UInt16 | Server port |

### 5) Capture point (where the Agent saw the request)

| Column | Type | Meaning |
|---|---|---|
| `observation_point` | String | The observation point the Agent captured from (e.g. c = client, s = server, or network points) |
| `capture_network_type_id` | UInt8 | Type of network captured from |
| `capture_nic_type` | UInt8 | Network card type |
| `capture_nic` | UInt32 | Network card ID |
| `signal_source` | UInt16 | Signal source (e.g. eBPF, network packets, or OTel), numeric |
| `tunnel_type` | UInt8 | Tunnel type, if any |
| `nat_source` | UInt8 | NAT data source |
| `direction_score` | UInt8 | Confidence score in determining the request direction (client/server) |
| `req_tcp_seq` | UInt32 | TCP sequence number of the request |
| `resp_tcp_seq` | UInt32 | TCP sequence number of the response |

### 6) Process and Syscall (eBPF)

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `gprocess_id_0` / `gprocess_id_1` | UInt32 | 全局客户端/服务端进程ID | Global process ID of the client / server |
| `process_id_0` / `process_id_1` | Int32 | 客户端/服务端进程ID | Process number (PID) of the client / server |
| `process_kname_0` / `process_kname_1` | String | 客户端/服务端系统进程 | System process name of the client / server |
| `syscall_trace_id_request` | UInt64 | SyscallTraceID-请求 | Syscall trace ID for the request; links requests across services without modifying code |
| `syscall_trace_id_response` | UInt64 | SyscallTraceID-响应 | Syscall trace ID for the response |
| `syscall_thread_0` / `syscall_thread_1` | UInt32 | Syscall线程-请求/响应 | The thread that executed the syscall for the request / response |
| `syscall_coroutine_0` / `syscall_coroutine_1` | UInt64 | Request/Response Syscall Coroutine | The coroutine for the request / response (e.g. Go) |
| `syscall_cap_seq_0` / `syscall_cap_seq_1` | UInt32 | Syscall序列号-请求/响应 | Syscall sequence number for the request / response |

### 7) Application protocol and record type

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `l7_protocol` | UInt8 | 0:未知 1:其他, 20:http1, 21:http2, 40:dubbo, 60:mysql, 80:redis, 100:kafka, 101:mqtt, 120:dns | Protocol code: 0 unknown, 1 other, 20 HTTP/1, 21 HTTP/2, 40 Dubbo, 60 MySQL, 80 Redis, 100 Kafka, 101 MQTT, 120 DNS |
| `biz_protocol` | String | 应用协议 | Application protocol name as text |
| `version` | String | 协议版本 | Protocol version |
| `type` | UInt8 | 日志类型, 0:请求, 1:响应, 2:会话 | Record type: 0 request, 1 response, 2 full session (request + response) |
| `is_tls` | UInt8 | - | Whether the connection is encrypted with TLS |
| `is_async` | UInt8 | - | Whether the request is asynchronous (Async) |
| `is_reversed` | UInt8 | - | Whether the request direction was reversed (the server initiated the connection) |

### 8) Request details

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `request_type` | String | 请求类型: HTTP方法、SQL命令类型、NoSQL、MQ、DNS查询类型 | Request type: e.g. GET/POST in HTTP, or SELECT in SQL |
| `request_domain` | String | 请求域名: HTTP主机名、RPC服务名、DNS查询域名 | Domain: the host name in HTTP, the service name, or the queried domain in DNS |
| `request_resource` | String | 请求资源: HTTP路径、RPC方法、SQL命令、NoSQL命令 | Requested resource: HTTP path or SQL command text |
| `endpoint` | String | 端点 | Endpoint |
| `request_id` | UInt64 (nullable) | 请求ID: HTTP、RPC、MQ、DNS | Request ID according to the protocol |
| `request_length` | Int64 (nullable) | 请求长度 | Request size in bytes |
| `x_request_id_0` / `x_request_id_1` | String | XRequestID0/1 | Value of the X-Request-ID header at the client / server |
| `http_proxy_client` | String | HTTP代理客户端 | The real client address behind an HTTP proxy |

### 9) Response details

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `response_status` | UInt8 | 响应状态 0:正常, 1:异常, 2:不存在, 3:服务端异常, 4:客户端异常 | Response status: 0 normal, 1 exception, 2 not found, 3 server error, 4 client error |
| `response_code` | Int32 (nullable) | 响应码: HTTP、RPC、SQL、MQ、DNS | Response code (e.g. 200 or 404 in HTTP) |
| `response_exception` | String | 响应异常 | Error or exception text |
| `response_result` | String | 响应结果, DNS解析地址 | Response result (e.g. the IP address resulting from DNS) |
| `response_duration` | UInt64 | - | Response time (usually in microseconds) |
| `response_length` | Int64 (nullable) | 响应长度 | Response size in bytes |
| `sql_affected_rows` | UInt64 (nullable) | sql影响行数 | Number of rows affected by the SQL command |
| `captured_request_byte` | UInt32 | - | Number of request bytes actually captured |
| `captured_response_byte` | UInt32 | - | Number of response bytes actually captured |

### 10) Distributed tracing

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `trace_id` | String | TraceID | Trace ID; groups all requests belonging to one operation |
| `_trace_id_2` | String | TraceID2 | Second trace ID (internal) |
| `trace_id_index` | UInt64 | TraceIDIndex | Internal index to speed up searching by `trace_id` |
| `span_id` | String | SpanID | ID of the current span |
| `parent_span_id` | String | ParentSpanID | ID of the parent span |
| `span_kind` | UInt8 (nullable) | SpanKind | Span kind (client/server/internal...) |
| `app_service` | String | app service | Service name |
| `app_instance` | String | app instance | Application instance name |
| `events` | String | OTel events | OpenTelemetry events attached to the span |

### 11) Business columns

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `biz_type` | UInt8 | Business Type | Business/activity type (numeric) |
| `biz_code` | String | - | Business code |
| `biz_scenario` | String | - | Scenario |
| `biz_response_code` | String | - | Response code at the business level |

### 12) Extra attributes and metrics (Key-Value)

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `attribute_names` | Array(String) | 额外的属性 | Names of the extra attributes |
| `attribute_values` | Array(String) | 额外的属性对应的值 | Attribute values (correspond to the names in order) |
| `metrics_names` | Array(String) | 额外的指标 | Names of the extra metrics |
| `metrics_values` | Array(Float64) | 额外的指标对应的值 | Metric values (correspond to the names in order) |

### Important notes

- **Most important columns for reading and analysis:** `start_time`, `l7_protocol`, `request_type`, `request_domain`, `request_resource`, `response_status`, `response_code`, `response_duration`, `trace_id`, `app_service`.
- **Response time:** `response_duration` has no comment, but in DeepFlow it is usually in microseconds, so divide it by 1000 for milliseconds (verify by comparing it with a request whose duration you know).
- **Time zone:** times are in Asia/Shanghai, and you are in Muscat (UTC+4), so the difference is 4 hours.
- **Compression columns** (T64, DoubleDelta) were omitted because they relate to internal storage.

---

# Columns of `flow_log.l4_flow_log`

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe flow_log.l4_flow_log" > l4_flow_log_columns.txt
```

The table is large (about 150 columns), so I split it into groups. As in `l7_flow_log`, the columns with the `_0` and `_1` suffix mean:

- `_0` = the first side (client / sender)
- `_1` = the second side (server / receiver)

The Chinese comments are translated, and the columns with no comment were explained from their names and what I know about DeepFlow.

> **Warning:** the output you sent was cut off at `fin_count`, and there may be columns after it. To be sure, run: `... > l4_flow_log_columns.txt` and then `wc -l l4_flow_log_columns.txt`. I explain only what arrived.

### 1) Identity and time

| Column | Type | Explanation |
|---|---|---|
| `_id` | UInt64 | Unique record ID |
| `time` | DateTime | Record time in seconds (v7.1.7.3 is the schema version) |
| `start_time` | DateTime64(6) | Flow start (microseconds) |
| `end_time` | DateTime64(6) | Flow end (microseconds) |
| `duration` | UInt64 | Flow duration (unit: microseconds) |
| `flow_id` | UInt64 | Flow ID (links it to `l7_flow_log`) |
| `aggregated_flow_ids` | String | IDs of the flows aggregated with it |
| `is_new_flow` | UInt8 | Whether it is a new flow (1) or a continuation (0) |
| `team_id` | UInt16 | Team ID |
| `agent_id` | UInt16 | ID of the Agent that captured the flow |

### 2) Link layer (L2)

| Column | Type | Explanation |
|---|---|---|
| `mac_0` / `mac_1` | UInt64 | MAC address of both sides |
| `eth_type` | UInt16 | Ethernet type (e.g. IPv4 or IPv6) |
| `vlan` | UInt16 | VLAN number |
| `l2_end_0` / `l2_end_1` | UInt8 | Whether the side is the L2 end (the real source/destination) |
| `l3_end_0` / `l3_end_1` | UInt8 | Whether the side is the L3 end |

### 3) Resources of both sides (`_0` / `_1` pairs)

Numeric IDs that are translated into names through `flow_tag`:

| Column (without suffix) | Type | Meaning |
|---|---|---|
| `region_id` | UInt16 | Region |
| `az_id` | UInt16 | Availability zone |
| `host_id` | UInt16 | Host server |
| `l3_device_type` / `l3_device_id` | UInt8 / UInt32 | Resource type and its ID |
| `pod_node_id` | UInt32 | Kubernetes node |
| `pod_ns_id` | UInt16 | Namespace |
| `pod_group_id` | UInt32 | Workload |
| `pod_id` | UInt32 | Pod |
| `pod_cluster_id` | UInt16 | Cluster |
| `l3_epc_id` / `epc_id` | Int32 | Virtual network (VPC) |
| `subnet_id` | UInt16 | Subnet |
| `service_id` | UInt32 | Service |
| `gprocess_id` | UInt32 | Global process ID |
| `tag_source` | UInt8 | Source of tag population |
| `province` | String | Province/region (from the geographic IP database) |

### 4) Automatic (Auto) fields

| Column | Type | Meaning |
|---|---|---|
| `auto_instance_id_0/1` | UInt32 | Instance ID for each side (prefers the Pod) |
| `auto_instance_type_0/1` | UInt8 | Instance type |
| `auto_service_id_0/1` | UInt32 | Service ID for each side (prefers the Service) |
| `auto_service_type_0/1` | UInt8 | Service type |

### 5) IP addresses, ports, and protocol

| Column | Type | Meaning |
|---|---|---|
| `ip4_0` / `ip4_1` | IPv4 | IPv4 address of both sides |
| `ip6_0` / `ip6_1` | IPv6 | IPv6 address of both sides |
| `is_ipv4` | UInt8 | Whether the address is IPv4 (1) or IPv6 (0) |
| `protocol` | UInt8 | Transport protocol (TCP=6, UDP=17) |
| `client_port` / `server_port` | UInt16 | Client / server port |
| `l7_protocol` | UInt8 | Identified application protocol |
| `request_domain` | String | Requested domain (if any) |

### 6) Tunnels

| Column | Type | Meaning |
|---|---|---|
| `tunnel_tier` | UInt8 | Tunnel level |
| `tunnel_type` | UInt16 | Tunnel type (e.g. VXLAN) |
| `tunnel_tx_id` / `tunnel_rx_id` | UInt32 | Tunnel ID on send / receive (e.g. VNI) |
| `tunnel_tx_ip4_0/1`, `tunnel_rx_ip4_0/1` | IPv4 | IPv4 addresses of the two tunnel endpoints |
| `tunnel_tx_ip6_0/1`, `tunnel_rx_ip6_0/1` | IPv6 | IPv6 addresses of the two tunnel endpoints |
| `tunnel_is_ipv4` | UInt8 | Whether the tunnel address is IPv4 |
| `tunnel_tx_mac_0/1`, `tunnel_rx_mac_0/1` | UInt32 | MAC addresses of the two tunnel endpoints |

### 7) Capture point

| Column | Type | Meaning |
|---|---|---|
| `observation_point` | String | The observation point the Agent captured from |
| `capture_network_type_id` | UInt8 | Type of network captured from |
| `capture_nic_type` / `capture_nic` | UInt8 / UInt32 | Network card type and ID |
| `signal_source` | UInt16 | Signal source (network packets, eBPF...) |
| `nat_source` | UInt8 | NAT data source |
| `nat_real_ip4_0/1` | IPv4 | Real IP address before NAT |
| `nat_real_port_0/1` | UInt16 | Real port before NAT |
| `direction_score` | UInt8 | Confidence score in determining the direction |
| `acl_gids` | Array(UInt16) | IDs of the matching ACL groups |
| `init_ipid` | UInt32 | Initial IP ID value |

### 8) Connection state (TCP)

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `close_type` | UInt16 | - | Reason the flow was closed (numeric: normal close, RST, timeout...) |
| `status` | UInt8 | 状态 0:正常, 1:异常, 2:不存在, 3:服务端异常, 4:客户端异常 | Flow status: 0 normal, 1 exception, 2 not found, 3 server error, 4 client error |
| `tcp_flags_bit_0/1` | UInt16 | - | Accumulated TCP flags for each side |
| `syn_seq` | UInt32 | 握手包的TCP SEQ序列号 | SEQ number of the handshake packet (SYN) |
| `syn_ack_seq` | UInt32 | 握手回应包的TCP SEQ序列号 | SEQ number of the handshake response packet (SYN-ACK) |
| `last_keepalive_seq` / `last_keepalive_ack` | UInt32 | - | Last SEQ / ACK of a keepalive packet |
| `syn_count` / `synack_count` | UInt32 | - | Number of SYN / SYN-ACK packets |
| `fin_count` | UInt32 | - | Number of FIN packets |

### 9) Data volume (Tx = sent, Rx = received)

| Column | Type | Meaning |
|---|---|---|
| `packet_tx` / `packet_rx` | UInt64 | Number of packets in the period |
| `byte_tx` / `byte_rx` | UInt64 | Number of bytes in the period |
| `l3_byte_tx` / `l3_byte_rx` | UInt64 | Network layer (L3) bytes |
| `l4_byte_tx` / `l4_byte_rx` | UInt64 | Transport layer (L4) bytes |
| `total_packet_tx` / `total_packet_rx` | UInt64 | Total packets since the start of the flow |
| `total_byte_tx` / `total_byte_rx` | UInt64 | Total bytes since the start of the flow |

### 10) L7 requests inside the flow

| Column | Type | Meaning |
|---|---|---|
| `l7_request` | UInt32 | Number of L7 requests |
| `l7_response` | UInt32 | Number of L7 responses |
| `l7_parse_failed` | UInt32 | Number of L7 protocol parsing failures |
| `l7_client_error` | UInt32 | Client errors (e.g. 4xx) |
| `l7_server_error` | UInt32 | Server errors (e.g. 5xx) |
| `l7_server_timeout` | UInt32 | Server timeouts |
| `l7_error` | UInt32 | Total errors |

### 11) Latency

| Column | Type | Original description | Explanation |
|---|---|---|---|
| `rtt` | Float64 | 单位: 微秒 | Total round-trip time (RTT) in microseconds |
| `rtt_client` | Float64 | 单位: 微秒 | The part of RTT between the capture point and the client |
| `rtt_server` | Float64 | 单位: 微秒 | The part of RTT between the capture point and the server |
| `tls_rtt` | Float64 | 单位: 微秒 | TLS handshake time |
| `srt_sum` / `srt_count` / `srt_max` | Float64 / UInt64 / UInt32 | max 微秒 (max in microseconds) | System Response Time: sum, count, max |
| `art_sum` / `art_count` / `art_max` | Float64 / UInt64 / UInt32 | max 微秒 (max in microseconds) | Application Response Time |
| `rrt_sum` / `rrt_count` / `rrt_max` | Float64 / UInt64 / UInt32 | max 微秒 (max in microseconds) | Request-Response Time |
| `cit_sum` / `cit_count` / `cit_max` | Float64 / UInt64 / UInt32 | max 微秒 (max in microseconds) | Client Idle Time |

Average = sum / count (e.g. `srt_sum / srt_count`).

### 12) Network quality

| Column | Type | Meaning |
|---|---|---|
| `retrans_tx` / `retrans_rx` | UInt32 | Number of retransmitted packets |
| `retrans_syn` / `retrans_synack` | UInt32 | SYN / SYN-ACK retransmissions |
| `zero_win_tx` / `zero_win_rx` | UInt32 | Times a TCP window of size zero appeared (the receiver is full) |
| `ooo_tx` / `ooo_rx` | UInt32 | Packets that arrived out of order |

### Important notes

- **Most important columns for analysis:** `start_time`, `ip4_0`, `ip4_1`, `server_port`, `protocol`, `byte_tx`, `byte_rx`, `rtt`, `retrans_tx`, `close_type`, `status`.
- **Difference from `l7_flow_log`:** here are the connection metrics (bytes, latency, retransmissions), while there are the request details (HTTP, SQL...).
- **Time zone:** times are in Asia/Shanghai, and you are in Muscat (UTC+4), so the difference is 4 hours.
- **Compression columns** (T64, Gorilla, DoubleDelta) were omitted because they relate to internal storage.

---

# Columns of `flow_log.l4_packet` (all 8 columns)

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe flow_log.l4_packet"
```

As expected, the table is small, and the output arrived in full (the last column is `packet_batch`). There are no official comments except on `time`, `start_time`, and `end_time`, so the rest of the explanation is based on the names and what I know about DeepFlow.

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 1 | `time` | DateTime | v7.1.7.3 | Record time in seconds (the number is the schema version) |
| 2 | `start_time` | DateTime64(6) | 精度: 微秒 = precision: microseconds | Start time of the captured packets |
| 3 | `end_time` | DateTime64(6) | 精度: 微秒 | End time of the captured packets |
| 4 | `flow_id` | UInt64 | - | Flow ID; links this table to `l4_flow_log` |
| 5 | `agent_id` | UInt16 | - | ID of the Agent that captured the packets |
| 6 | `team_id` | UInt16 | - | Team ID |
| 7 | `packet_count` | UInt32 | - | Number of packets in this batch |
| 8 | `packet_batch` | String | - | The raw packets themselves stored as a batch (binary data) |

### The idea

This table has no resource columns (pod, node...) and no IP. All it has is:

- From which flow? → `flow_id`
- When? → `start_time` / `end_time`
- How many packets? → `packet_count`
- The packets themselves → `packet_batch`

---

# Columns of `flow_log.l7_packet` (all 9 columns)

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "describe flow_log.l7_packet"
```

The table is small and very similar to `l4_packet`, and the output arrived in full (the last column is `team_id`). There are no official comments except on `time`, `start_time`, `end_time`, and `packet_batch`, so the rest of the explanation is based on the names and what I know about DeepFlow.

| # | Column | Type | Original description | Explanation |
|---|---|---|---|---|
| 1 | `time` | DateTime | v7.1.7.3 | Record time in seconds (the number is the schema version) |
| 2 | `start_time` | DateTime64(6) | 精度: 微秒 = precision: microseconds | Start time of the captured packets |
| 3 | `end_time` | DateTime64(6) | 精度: 微秒 | End time of the captured packets |
| 4 | `flow_id` | UInt64 | - | Flow ID; links this table to `l7_flow_log` and `l4_flow_log` |
| 5 | `agent_id` | UInt16 | - | ID of the Agent that captured the packets |
| 6 | `packet_count` | UInt32 | - | Number of packets in this batch |
| 7 | `packet_batch` | String | data format reference: link to the PCAP format reference (IETF draft) | The raw packets themselves stored as a batch in PCAP format (binary data) |
| 8 | `acl_gids` | Array(UInt16) | - | IDs of the ACL groups (policies) that matched these packets |
| 9 | `team_id` | UInt16 | - | Team ID |