## 🔵 Step 0 — Start Here

```bash
GET _cluster/health
```

Check:

- `status` → green / yellow / red
- `unassigned_shards`
- `active_shards_percent_as_number`

---

# 🟡 CASE 1 — **Cluster = YELLOW**

Problem: **Replica shards unassigned**

---

## ➤ Step 1 — Why?

```bash
GET _cluster/allocation/explain
```

---

## ➤ Step 2 — Apply Fix Based on Reason

### A. Single Node Cluster (MOST COMMON IN EXAM)

**Symptom:**

> cannot allocate replica shard, only one node

✔ Fix:

```bash
PUT /*/_settings
{
  "number_of_replicas": 0
}
```

---

### B. Not Enough Nodes

**Symptom:**

> not enough nodes to allocate replicas

✔ Fix:

- Add nodes
  **OR**

```bash
PUT index/_settings
{
  "number_of_replicas": 0
}
```

---

### C. Allocation Disabled

```bash
GET _cluster/settings
```

✔ Fix:

```bash
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

---

### D. Disk Watermark Exceeded

**Symptom:**

> disk usage exceeded

✔ Fix:

- Free disk
  **OR (temporary exam fix):**

```bash
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%"
  }
}
```

---

### E. Shard Allocation Filtering

**Symptom:**

> node does not match index routing rules

✔ Fix:

```bash
GET index/_settings
```

Remove bad filters:

```bash
PUT index/_settings
{
  "index.routing.allocation.include._name": null
}
```

---

# 🔴 CASE 2 — **Cluster = RED**

Problem: **Primary shards unassigned (CRITICAL)**

---

## ➤ Step 1 — Diagnose

```bash
GET _cluster/allocation/explain
```

---

## ➤ Step 2 — Common Causes + Fixes

---

### A. Missing Data / Node Loss

✔ Fix (recover from snapshot):

```bash
POST _snapshot/repo/snap/_restore
```

---

### B. Corrupted / Unrecoverable Primary

✔ Force allocate:

```bash
POST _cluster/reroute
{
  "commands": [
    {
      "allocate_stale_primary": {
        "index": "my-index",
        "shard": 0,
        "node": "node-1",
        "accept_data_loss": true
      }
    }
  ]
}
```

Exam note: **Only if explicitly required**

---

### C. Allocation Disabled / Disk Issues

👉 Same fixes as YELLOW case

---

# CASE 3 — **Unassigned Shards (General)**

```bash
GET _cat/shards?v
```

Look for:

- `UNASSIGNED`
- `INITIALIZING`
- `RELOCATING`

---

# Snapshot Troubleshooting

## List snapshots:

```bash
GET _snapshot/repo/_all
```

## Restore:

```bash
POST _snapshot/repo/snapshot_01/_restore
{
  "indices": "index_name"
}
```

---

# 🔁 CASE 4 — **Rebalancing / Stuck Shards**

## Check:

```bash
GET _cluster/settings
```

## Fix:

```bash
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.rebalance.enable": "all"
  }
}
```

---

# GOLDEN EXAM SHORTCUTS

If you’re under time pressure:

| Problem             | Default Fix                      |
| ------------------- | -------------------------------- |
| Yellow cluster      | 👉 `number_of_replicas: 0`       |
| Red cluster         | 👉 restore snapshot OR reroute   |
| Unassigned shards   | 👉 `_cluster/allocation/explain` |
| Disk issue          | 👉 increase watermark            |
| Allocation disabled | 👉 enable allocation             |

---

# Common Exam Traps

- ❌ Increasing replicas when already failing
- ❌ Ignoring `_cluster/allocation/explain`
- ❌ Using wrong index name
- ❌ Forgetting `accept_data_loss` in reroute
- ❌ Fixing symptom instead of cause

---

# Mental Model

> **Health → Explain → Fix Root Cause**

---
