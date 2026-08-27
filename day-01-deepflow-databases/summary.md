# Databases in the DeepFlow Project

> Technical documentation explaining the data storage architecture of the [DeepFlow](https://github.com/deepflowio/deepflow) project — prepared while learning and training on the project, aimed at helping any new developer joining the project understand where each type of data is stored and why.

---

## 1. Overview

DeepFlow relies on **two main databases**, each with a completely different role:

| Database | Type | Role | Where it lives in the code |
|---|---|---|---|
| **ClickHouse** | Columnar, real-time analytics database (OLAP) | Stores actual monitoring data: Flows, Metrics, Logs, Traces, Events, Profiles | `server/ingester` (writing) and `server/querier` (reading) |
| **MetaDB (MySQL/MariaDB)** | Relational database via the `gorm` ORM | Stores metadata: cloud resources, system configuration, agent information, tag-translation dictionaries | `server/controller/db/metadb` |

**Core idea:** ClickHouse stores "the event itself" (e.g., an HTTP request, a network packet, a performance metric), while MetaDB stores "information about resources" (e.g., which VM this IP belongs to, which cluster this Pod belongs to) — information that is later used to translate the numeric IDs in ClickHouse data into human-readable names.

There is also alternative support for **PostgreSQL** and **DM (Dameng)** as alternate MetaDB backends behind the same abstraction layer (`sqladapter`), but MySQL/MariaDB is the default.

---

## 2. The Core File Behind ClickHouse's Structure

Before diving into the tables, here's the "engine" that builds everything:

- **`server/libs/ckdb/ckdb.go`** — defines the core types: `ColumnType`, `EngineType`, `CodecType`, `IndexType`, and the `Table`/`Column` structs. This is the generic type system used everywhere.
- **`server/libs/ckdb/table.go`** — defines `ckdb.Table` (Database, LocalName, GlobalName, Columns, Engine, TTL, PartitionFunc, OrderKeys...) and the `makeLocalTableCreateSQL` function that actually generates the `CREATE TABLE` statements. Every sub-database calls into this file to create its tables.

### Key concepts that repeat across every ClickHouse database:
- **Local vs. Distributed/Global tables**: every table has a local version (`_local`) and a distributed/global version.
- **AggregationInterval**: data is aggregated at intervals (1 second / 1 minute / 1 hour / 1 day).
- **Multi-org support**: an `OrgDatabasePrefix` prefix supports multiple organizations (tenants) on the same cluster.
- **The flow_tag pattern**: nearly every sub-database has two helper tables named `<name>_field` and `<name>_field_value`, used as a "tag dictionary" to interpret the data.

---

## 3. ClickHouse Databases in Detail

### 3.1 Full Summary Table (Database → Tables → Writer → Reader)

| Database | Main Tables | Writer | Reader |
|---|---|---|---|
| **flow_log** | `l4_flow_log`, `l7_flow_log`, `l4_packet`, `l7_packet`, `trace_tree`, `span_with_trace_id`, `_field`/`_field_value` | `server/ingester/flow_log`, `server/ingester/pcap` | `server/querier` (`DB_NAME_FLOW_LOG`) |
| **flow_metrics** | `network.1m/1s`, `network_map.1m/1s`, `application.1m/1s`, `application_map.1m/1s`, `traffic_policy.1m` | `server/ingester/flow_metrics` | `DB_NAME_FLOW_METRICS` |
| **event** | `event`, `file_event`, `file_agg_event`, `file_mgmt_event`, `proc_perm_event`, `proc_ops_event`, `alert_event`, `alert_record`, `file_event_metrics`, `_field`/`_field_value` | `server/ingester/event` | `DB_NAME_EVENT` |
| **ext_metrics** | `metrics` (alias `ext_common`), `_field`/`_field_value` | `server/ingester/ext_metrics` | `DB_NAME_EXT_METRICS` |
| **deepflow_admin** | `deepflow_server` | `server/ingester/ext_metrics` | `DB_NAME_DEEPFLOW_ADMIN` |
| **deepflow_tenant** | `deepflow_collector` | `server/ingester/ext_metrics` | `DB_NAME_DEEPFLOW_TENANT` |
| **prometheus** | `samples`, `prometheus_field`/`_value` | `server/ingester/prometheus` | `DB_NAME_PROMETHEUS` + `trans_prometheus` |
| **profile** | `in_process`, `in_process_metrics`, `_field`/`_field_value` | `server/ingester/profile` | `DB_NAME_PROFILE` |
| **application_log** | `log`, `log_field`/`_field_value` | `server/ingester/app_log` | `DB_NAME_APPLICATION_LOG` |
| **flow_tag** | (virtual: `_field`/`_field_value` tables from every database) | `server/ingester/flow_tag` (shared library) | `DB_NAME_FLOW_TAG` |
| **deepflow_system** | `deepflow_system` (legacy/abandoned) | No active writer | Referenced only in `datasource/handle.go` |

### 3.2 Details of Each Database

#### 📁 `flow_log` — Raw Trace Data
Stores raw network data and distributed tracing:
- `l4_flow_log` — Layer 4 data, ordered by `time, l3_epc_id_1, ip4_1, ip6_1, server_port`
- `l7_flow_log` — Layer 7 data (HTTP, gRPC...), adds `l7_protocol` as an order key
- `l4_packet` / `l7_packet` — raw packets
- `trace_tree` / `span_with_trace_id` — distributed tracing data — the most important tables for understanding DeepFlow's "Zero Code Tracing" concept

**Reference files:** `server/ingester/flow_log/dbwriter/flowlog_writer.go`, `tracetree_writer.go`, `span_writer.go`

#### 📁 `flow_metrics` — Aggregated Metrics
Performance metrics (bytes, packets, rtt, retrans...) aggregated at 1-second/1-minute intervals:
- `network.*` / `network_map.*` — network-level metrics
- `application.*` / `application_map.*` — application-level metrics
- `traffic_policy.1m` — related to ACL policies

**Note:** Legacy names are still used in some code paths: `vtap_flow`, `vtap_wan`, `vtap_packet`, `vtap_acl`, `vtap_app`

#### 📁 `event` — Events and Alerts
- `event` — combines Resource Events and Kubernetes Events together
- `file_event` and its family — events specific to files and processes
- `alert_event` / `alert_record` — alerts

#### 📁 `ext_metrics` — General External Metrics
A generic table (`metrics`) for storing any external key/value metrics, plus tables for DeepFlow's own status (`deepflow_admin`, `deepflow_tenant`).

#### 📁 `prometheus` — Prometheus Integration
- `samples` — the actual values, ordered by `metric_id, time, target_id`
- `app_label_value_id_%d` columns are added dynamically (via ALTER TABLE) as label cardinality grows

#### 📁 `profile` — Performance Profiling Data
The `in_process` table stores CPU/Heap profiling data — tied to DeepFlow's "Continuous Profiling" feature.

#### 📁 `application_log` — Application Logs
The `log` table supports full-text search via `tokenbf` indexing on the `body` field, plus Bloom Filter indexing on `trace_id`/`span_id` — directly linking logs to traces.

#### 📁 `flow_tag` — Shared Dictionary
Not an independent database with its own writer, but a "logical view" over the `_field`/`_field_value` tables that exist inside each of the databases above.

---

## 4. The MetaDB Database (MySQL/MariaDB)

### 4.1 General Information
| Property | Value |
|---|---|
| Default database type | MySQL / MariaDB |
| Library used | `gorm` (ORM) + `github.com/go-sql-driver/mysql` |
| Alternate backends | PostgreSQL, DM (Dameng) |
| Default database name | `deepflow` |
| Naming strategy | Singular table names instead of plural, via `NamingStrategy{SingularTable: true}` |
| Multi-org support | Each organization gets a separate database, e.g. `0002_deepflow` |
| Version tracking table | `db_version` — compares the current version against `DB_VERSION_EXPECTED` and automatically runs SQL upgrades (ISSU) |

### 4.2 Table Distribution by File

| File | Struct Count | Category |
|---|---|---|
| `model/model.go` | 36 | Controller/Analyzer/Agent(vtap)/ACL/NPB/Plugin/Org/Team/general config |
| `model/prometheus_model.go` | 10 (7 real tables + 3 helpers) | Prometheus metric/label name/value encoding |
| `model/ch_model.go` | 54 (~50 real tables) | `ch_*` tables for tag translation in queries |
| `model/platform_rsc_model.go` | 51 (~48 real tables) | Cloud/Kubernetes resource inventory |
| `agent_config/db.go` | 3 | Agent group YAML configuration |

### 4.3 Cloud/Kubernetes Resource Tables (Most Important) — `platform_rsc_model.go`

| Struct | Table Name | Description |
|---|---|---|
| `Domain` | `domain` | A top-level cloud domain (one cloud account/integration) |
| `SubDomain` | `sub_domain` | A Kubernetes sub-domain under a Domain |
| `Host` | `host_device` | The hypervisor host |
| `VM` | `vm` | Virtual machine |
| `VPC` | `epc` | Virtual private cloud |
| `Network` / `Subnet` | `vl2` / `vl2_net` | Network and its subnet |
| `VInterface` | `vinterface` | Virtual network interface (vNIC) |
| `LANIP` / `WANIP` | `vinterface_ip` / `ip_resource` | Internal/external IP addresses |
| `LB` / `LBListener` / `LBTargetServer` | `lb` / `lb_listener` / `lb_target_server` | Load balancer and its components |
| `PodCluster` | `pod_cluster` | Kubernetes cluster |
| `PodNamespace` | `pod_namespace` | Namespace |
| `PodNode` | `pod_node` | Kubernetes node |
| `PodService` / `PodServicePort` | `pod_service` / `pod_service_port` | Kubernetes Service and its ports |
| `PodGroup` | `pod_group` | Deployment/StatefulSet/RC |
| `Pod` | `pod` | The Pod itself |
| `ConfigMap` | `config_map` | Kubernetes configuration |
| `CustomService` | `custom_service` | A user-defined "custom" service |

**Who consumes these tables:**
- `cloud/{aliyun,huawei,tencent,baidubce,kubernetes_gather,...}` — gather data from cloud providers
- `recorder/*` — the sync engine (diffs and updates the tables)
- `http/service` and `http/router` — REST API interfaces
- `trisolaris/*` — distributes data down to agents
- `tagrecorder/*` — builds the `ch_*` tables from these tables

### 4.4 Controller and Operational Tables — `model.go`

| Struct | Table Name | Description |
|---|---|---|
| `Controller` | `controller` | Main controller server |
| `Analyzer` | `analyzer` | Analyzer server |
| `VTap` | `vtap` | The Agent (previously named VTap) |
| `VTapGroup` | `vtap_group` | Group of agents |
| `DataSource` | `data_source` | A data source with a retention period |
| `ACL` | `acl` | Access control rules |
| `NpbPolicy` / `NpbTunnel` | `npb_policy` / `npb_tunnel` | Packet broker distribution policies and tunnels |
| `AlarmPolicy` | `alarm_policy` | Alerting policies |
| `ORG` / `Team` / `User` | `org` / `team` / `user` | User and organization management |

### 4.5 Tag-Dictionary Tables (`ch_*`) — `ch_model.go`

These tables (roughly 50 of them) are the "dictionary" that translates the numeric IDs in ClickHouse data (like `l3_epc_id` or `pod_id`) into human-readable names when results are displayed. The most important ones:

| Struct | Table Name | Description |
|---|---|---|
| `ChRegion` | `ch_region` | Region names |
| `ChAZ` | `ch_az` | Availability zones |
| `ChVPC` | `ch_l3_epc` | Names of private networks |
| `ChDevice` | `ch_device` | A comprehensive table for the names of all device types |
| `ChPod` / `ChPodCluster` / `ChPodNamespace` / `ChPodNode` | `ch_pod*` | Translation of Kubernetes resources |
| `ChIPResource` | `ch_ip_resource` | A wide table linking an IP to all possible resources |
| `ChPrometheusLabelName` / `ChPrometheusMetricName` | `ch_prometheus_*` | Translation of Prometheus labels |

**Who consumes them:** almost exclusively `tagrecorder/*` (roughly one Go file per table), in addition to the querier engine, which uses them to translate the IDs in ClickHouse query results.

### 4.6 Prometheus Metadata Tables — `prometheus_model.go`

A custom encoding system that converts textual label/metric names and values into compact numeric IDs (to save space in ClickHouse):

| Struct | Table Name |
|---|---|
| `PrometheusMetricName` | `prometheus_metric_name` |
| `PrometheusLabelName` | `prometheus_label_name` |
| `PrometheusLabelValue` | `prometheus_label_value` |
| `PrometheusMetricTarget` | `prometheus_metric_target` |

---

## 5. Summary: How Do the Two Databases Work Together?

```
        ┌─────────────────────┐
        │    Data Sources      │
        │  (Agent via eBPF)     │
        └──────────┬───────────┘
                    │
        ┌───────────▼────────────┐
        │   ClickHouse             │   ← Massive time-series event/metric data
        │ (flow_log, flow_metrics, │
        │  event, prometheus...)   │
        └───────────┬────────────┘
                    │ IDs only (no names)
                    │ translated at read time via
        ┌───────────▼────────────┐
        │   MetaDB (MySQL)          │   ← Metadata about resources
        │ (domain, vm, pod,         │
        │  ch_* dictionary...)      │
        └──────────────────────────┘
```

This design is why DeepFlow performs so well: massive data (Flows/Metrics) is stored as compact IDs in ClickHouse, and translated into human-readable names only at read time via the `ch_*` tables in MetaDB — instead of repeating long text strings in every single record.

---

## 6. Key Reference Files (for later use)

| Purpose | Path |
|---|---|
| General ClickHouse engine | `server/libs/ckdb/ckdb.go`, `table.go` |
| flow_metrics schema | `server/libs/flow-metrics/tag.go`, `flow_meter.go`, `app_meter.go` |
| flow_log schema | `server/ingester/flow_log/dbwriter/` |
| event schema | `server/ingester/event/dbwriter/` |
| prometheus schema | `server/ingester/prometheus/dbwriter/` |
| Comprehensive retention reference | `server/ingester/datasource/handle.go` |
| Querier database/table registry | `server/querier/engine/clickhouse/common/const.go` |
| MetaDB system (ORM) | `server/controller/db/metadb/session/` |
| Migration system | `server/controller/db/metadb/migrator/` |
| Cloud resources (Model) | `server/controller/db/metadb/model/platform_rsc_model.go` |
| Controller configuration (Model) | `server/controller/db/metadb/model/model.go` |
| Tag dictionary (Model) | `server/controller/db/metadb/model/ch_model.go` |

---

## 7. Why Did the Developers Choose This Approach? (The Reasoning)

Now that the architecture is clear, the natural question is: why did they design the system this way (two separate databases + an ID system + deferred translation)? There are clear engineering reasons behind every decision:

### 7.1 Why separate the massive data (ClickHouse) from the metadata (MetaDB)?
- **The data volumes are vastly different**: Flow and Metric data is generated at a massive rate (millions of records per second from dozens of agents), while resource data (VM, Pod, Network...) changes relatively slowly. Using the same database for both would create a bottleneck.
- **The query patterns are different**: massive data needs fast analytical reads across millions of rows (OLAP), while resource data needs precise, frequent CRUD operations (OLTP). Each database type is designed for a different workload from the ground up.

### 7.2 Why ClickHouse specifically for monitoring data?
- A **columnar database** built for real-time analytics, capable of processing hundreds of millions of rows per second on a single server.
- Supports **smart indexing** (Bloom filters on trace_id/span_id, tokenbf for full-text search) that speeds up searching through massive volumes of records.
- Supports **automatic aggregation (Materialized Views)** at different intervals (1s/1m/1h/1d), so averages don't need to be recomputed manually every time.

### 7.3 Why an ID system + translation tables (`ch_*`) instead of storing names directly?
This is the most important design decision in the project, and the reason is purely technical:
- If they stored the full name (e.g., a Pod's or Namespace's name) in every one of the millions of flow records, the same text would repeat millions of times and waste enormous storage space.
- Instead, they store only a **compact ID** in the ClickHouse data, and translate it into a readable name only at read time via the `ch_*` tables in MetaDB.
- This approach (which the project calls **SmartEncoding**) is a direct reason for DeepFlow's superior performance, significantly reducing storage size compared to storing full text or the traditional low-cardinality methods used in ClickHouse.

### 7.4 Why MySQL/MariaDB (and not NoSQL, for instance) for metadata?
- Resource data (VM ↔ Network ↔ VPC ↔ Domain...) has **complex, interconnected relationships** between tables, and this is exactly where relational databases outperform NoSQL.
- Using a **mature ORM** like `gorm` makes migrations and version management easier via the `db_version` table — critical for an open-source project that evolves continuously and needs backward compatibility.
- Supporting alternatives like PostgreSQL and DM gives organizations flexibility based on their existing environment, without changing the application logic (thanks to the `sqladapter` layer).

### 7.5 Why separate Local and Distributed tables in ClickHouse?
- This is a standard pattern in distributed systems: the local table (`_local`) actually stores data on each node, while the distributed/global table provides a unified query interface that aggregates data across all nodes.
- This allows the project to scale horizontally — you add more nodes without changing the query structure.

### 7.6 Why a separate database per organization (Multi-org)?
- Strict tenant isolation at the database level itself (not just the row level), providing higher security and better performance when multiple customers share the same system (SaaS model).

### 7.7 The Bigger Picture: Why All This Complexity in the First Place?
DeepFlow follows a **"Zero Code" philosophy via eBPF** — meaning it automatically collects monitoring data from the kernel without the developer adding any code inside their application. This decision automatically generates an enormous amount of raw data (since every network packet and every function call is captured), so a storage architecture capable of handling this volume efficiently had to be built — and this is exactly what explains all the decisions described above.

---

*Last updated: this report was prepared during self-directed learning on the DeepFlow project in preparation for onboarding/training.*