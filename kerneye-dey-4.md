# Day 4: The Overview page 🧭

## The goal

When an analyst opens KernEye, they should see in one second whether the
system is running correctly. If something is red, they click it and see why.

---

## The idea 💡

One page called **Overview**: one verdict at the top, then 20 indicators in
4 sections.

| Section | Indicators |
|---|---|
| 🔐 Security | Critical alerts, Failed logins, Suspicious commands, Sensitive file access, Unusual outbound traffic |
| 🌐 Network | Open ports, Data in / out, Connections, Succeeded / failed, Closed cleanly / with reset |
| 🖥️ Resources | CPU, Memory, Disk space, System load average, Swap |
| ⚡ Performance | Disk I/O, Throughput, Error rates, Process health, Hardware health |

---

## What I did today 🔧

1. Checked which data I already had. 6 indicators had none.
2. Changed the agent to collect it: load, swap, disk space, temperature, Wi-Fi.
3. Built the Overview page with a verdict banner and 20 tiles.
4. Made Failed logins live, using SSH logs.
5. Cleaned the Threat Monitoring page.
6. Pushed everything to GitHub.

---

## The rule 🛡️

**The page must never lie.**

- Everything normal → "Everything we monitor is normal"
- A data source stops → "Cannot confirm everything is fine"
- No collector yet → gray tile "Not collected yet"

A missing number is never shown as 0.

---

## Bugs I caught 🐛

- Memory was counted with 1000 instead of 1024.
- The agent ignored Wi-Fi, so traffic showed 0.
- An alert query would have hidden critical alerts.
- One failed login produced 10 log rows, so the count was 10 times too high.

---

## Status 📊

**16 of 20 indicators work.** Four are missing:

| Missing | Why |
|---|---|
| Suspicious commands | Needs new eBPF code |
| Sensitive file access | Needs new eBPF code |
| Closed cleanly / with reset | Needs new eBPF code |
| Unusual outbound traffic | Needs days of data to know what is normal |

They show as gray tiles, so nobody thinks they are fine.

---

## Today's discovery 🔎

My first deploy rolled back automatically. The code was fine. The test
server has a slow disk, and the server needs over a minute to start. With a
3-minute wait it worked.

---

## Next step 🚀

Merge the work on GitHub, then add the eBPF indicators one by one, starting
with Closed / with reset.

## Question for you ❓

Should Overview be the first page analysts see when they log in?