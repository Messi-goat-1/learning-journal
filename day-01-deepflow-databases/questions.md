# Day 1 — Questions: DeepFlow Databases

Work through this **one stage at a time**. Read the matching section of the summary report, then open the code and answer from what you actually see — don't move to the next stage until you're done with the current one.

---

## 🔹 Stage 1: The General ClickHouse Engine
*Read section "2" of the report, then open the code and answer:*

1. Open `server/libs/ckdb/table.go` — what fields exist in `struct Table`?
2. What's the actual difference in the code between `LocalName` and `GlobalName`?
3. Find the function `makeLocalTableCreateSQL` — roughly what does the SQL it generates look like?

---

## 🔹 Stage 2: ClickHouse Sub-Databases (flow_log, flow_metrics, event...)
*Read section "3" in full, then:*

4. Open `server/ingester/flow_log/dbwriter/flowlog_writer.go` — what's the name of the function responsible for writing a record into `l4_flow_log`?
5. Open `server/ingester/event/dbwriter/event_writer.go` (or `event.go`) — how do they distinguish a regular `event` from an `alert_event`?
6. Open `server/ingester/prometheus/dbwriter/prometheus_writer.go` — find the logic that dynamically adds columns (ALTER TABLE) — what triggers it?

---

## 🔹 Stage 3: MetaDB and Cloud Resources
*Read sections "4.1" through "4.4", then:*

7. Open `server/controller/db/metadb/model/model.go` — find `struct VTap` — how many fields does it have? Which field indicates the Agent's status?
8. Open `server/controller/db/metadb/model/platform_rsc_model.go` — find `struct Pod` — which fields link it to `PodNamespace` and `PodCluster`?
9. Open `server/controller/db/metadb/migrator/migrator.go` — what exactly happens if `db_version` is lower than `DB_VERSION_EXPECTED`?

---

## 🔹 Stage 4: Translation Tables (`ch_*`)
*Read section "4.5", then:*

10. Open `server/controller/db/metadb/model/ch_model.go` — find `struct ChDevice` — what columns does it have?
11. Open any file under `tagrecorder/` (e.g. `ch_device.go`) — how does data flow from `platform_rsc_model` into `ch_model`?

---

## 🔹 Stage 5: The Engineering Reasons (section "7")
*This stage is about thinking, not code:*

12. After everything you've seen in the code, try explaining in your own words: why do they store `device_id` instead of `device_name` in ClickHouse?
13. In your opinion, if they merged the two databases (ClickHouse + MetaDB) into one, what problems would that cause?

---

💡 **Reminder:** you don't have to finish all 5 stages in one day. Pick one stage, work through it slowly, and write down what you actually found in your answers file — even if it's incomplete or wrong at first.