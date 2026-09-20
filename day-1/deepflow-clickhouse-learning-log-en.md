## 📅 Today: Exploring ClickHouse Databases in DeepFlow

### 🎯 Goal
Understand the data stored in the DeepFlow system so I can write queries with confidence.

### 🛠️ What I Did
I ran the following command inside the Pod to list the databases:

```bash
kubectl exec -n deepflow deepflow-clickhouse-0 -- clickhouse-client -q "show databases"
```

Some of them belong to DeepFlow, and the rest come by default with ClickHouse.

✅ **Progress:** I extracted the important columns from these databases to use when writing queries.

### 📚 What I Learned

| Database | What It Stores |
|---|---|
| `flow_log` | Flow logs (L4 and L7) and tracing data |
| `application_log` | Application logs |
| `event` | Resource events and changes |
| `ext_metrics` | External metrics (Telegraf and OpenTelemetry) |
| `prometheus` | Prometheus metrics |
| `deepflow_admin` / `deepflow_tenant` | Operational data specific to DeepFlow itself |

### ⏭️ Next Steps
- **Coming days:** Extract the columns from `flow_tag` and `profile`. I'll take it step by step and not rush, because too much data at once distracts me. The priority is to understand it properly before moving on to building the UI.
- **Tomorrow:** Learn the API and pull data from it, then build a simple frontend with a simple backend.

---

**Short version:**

> Today I explored the ClickHouse databases in DeepFlow, understood the role of each one, and extracted their key columns for writing my queries. Coming days: `flow_tag` and `profile`. Tomorrow: the API and a simple frontend with a backend. 🚀