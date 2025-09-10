# CephFS on `rootfs` — Setup Log & Explanations

This document records the exact commands executed to create a **CephFS** filesystem whose data and metadata pools are **restricted to the `rootfs` CRUSH root** (OSDs 0–2, class `hdd`). Each command includes a short explanation and, where provided, the actual output.

> **Context:** Your cluster has two CRUSH roots: `roothdfs` (OSDs 3–5) and `rootfs` (OSDs 0–2). The goal is to build a CephFS named `cephfs` using only the `rootfs` branch.

---

## 1) Create a CRUSH Rule Scoped to `rootfs` (HDD)

```bash
ceph osd crush rule create-replicated rootfs_hdd_rule rootfs host hdd
```

**What it does:**  
Creates a replicated CRUSH rule named `rootfs_hdd_rule` that:
- **Starts at** the CRUSH **root** `rootfs` → placement is confined to that root (OSDs 0–2 in your tree).  
- Uses **host** as the failure domain → replicas are spread across different hosts.  
- Restricts placement to device **class `hdd`** → objects land only on HDD OSDs within `rootfs`.

---

## 2) Create CephFS Pools (Metadata & Data) and Bind the Rule

### 2.1 Metadata pool

```bash
ceph osd pool create cephfs_metadata 32 32
```

**What it does:**  
Creates a pool named `cephfs_metadata` with initial **PG/PGP = 32/32**. (Autoscaler may later adjust).

```bash
ceph osd pool set cephfs_metadata crush_rule rootfs_hdd_rule
# Output:
# set pool 2 crush_rule to rootfs_hdd_rule
```

**What it does:**  
Pins the `cephfs_metadata` pool to the `rootfs_hdd_rule`, ensuring metadata stays on the `rootfs` branch (HDDs).

### 2.2 Data pool

```bash
ceph osd pool create cephfs_data 64 64
```

**What it does:**  
Creates `cephfs_data` with **PG/PGP = 64/64**. (Again, autoscaler may tune this.)

```bash
ceph osd pool set cephfs_data crush_rule rootfs_hdd_rule
# Output:
# set pool 3 crush_rule to rootfs_hdd_rule
```

**What it does:**  
Pins the data pool to the same `rootfs` HDD-only rule.

---

## 3) Create the CephFS Filesystem

```bash
ceph fs new cephfs cephfs_metadata cephfs_data
```

**What it does:**  
Creates a **CephFS** named **`cephfs`**, using the metadata pool `cephfs_metadata` and the data pool `cephfs_data`.

**Output (important warning & success):**
```
Pool 'cephfs_data' (id '3') has pg autoscale mode 'on' but is not marked as bulk.
Consider setting the flag by running
  # ceph osd pool set cephfs_data bulk true
new fs with metadata pool 2 and data pool 3
```

**Meaning:**  
The autoscaler sees `cephfs_data` as a large/“bulk” pool and suggests adding the **bulk** hint (see step 5).

---

## 4) Deploy the MDS (Metadata Server)

```bash
ceph orch apply mds cephfs --placement="master"
# Output:
# Scheduled mds.cephfs update...
```

**What it does:**  
Schedules an MDS daemon for the filesystem `cephfs` on host `master`. CephFS needs an MDS to serve directory operations.

---

## 5) Mark the Data Pool as Bulk (Autoscaler Hint)

```bash
ceph osd pool set cephfs_data bulk true
# Output:
# set pool 3 bulk to true
```

**What it does:**  
Sets the **bulk** flag on `cephfs_data`. This is a hint to the **PG autoscaler** that this pool is expected to hold **large amounts of data**. The autoscaler will assign PGs more proportionally to its size, and avoid over-allocating PGs to small pools (like metadata).

**Verify autoscaler view:**

```bash
ceph osd pool autoscale-status
```

**Sample output from your run:**
```
POOL               SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
cephfs_metadata  32768                 3.0         2699G  0.0000                                  4.0      32              on         False
cephfs_data          0                 3.0         2699G  0.0000                                  1.0      64              on         True
```

**How to read it:**
- `AUTOSCALE on` → autoscaler can adjust PGs as usage grows.  
- `BULK True` for `cephfs_data` (good); `BULK False` for metadata (expected).  
- `PG_NUM` shows current PGs; `NEW PG_NUM` would show suggested changes if any.

---

## 6) Basic Health & Placement Checks

### Check replication size
```bash
ceph osd pool get cephfs_data size
# Output:
# size: 3
```
**What it does:**  
Confirms replication factor for `cephfs_data` (default **size=3**, **min_size=2** is typical for 3 OSDs). Repeat for metadata if desired.

### Filesystem/MDS status
```bash
ceph fs status
```
**Your output:**
```
cephfs - 0 clients
======
RANK  STATE           MDS              ACTIVITY     DNS    INOS   DIRS   CAPS
 0    active  cephfs.master.ekdxsn  Reqs:    0 /s    10     13     12      0
      POOL         TYPE     USED  AVAIL
cephfs_metadata  metadata  96.0k   854G
  cephfs_data      data       0    854G
MDS version: ceph version 18.2.7 (6b0e9...) reef (stable)
```

**Meaning:**  
The filesystem is created, an MDS is active, metadata pool has minimal usage, and data pool is empty and available.

---

## Notes & Best Practices

- **Why “bulk” on data?** CephFS data grows large; the autoscaler should prioritize PGs for it. Metadata pools are small but latency-sensitive, so they **should not** be marked bulk.  
- **CRUSH rule scope:** By using `rootfs_hdd_rule`, all pool objects stay on OSDs **within `rootfs`** and **class `hdd`**, enforcing physical/data isolation from `roothdfs`.  
- **Failure domain `host`:** Ensures replicas land on different hosts for better availability.  
- **Autoscaler:** You can optionally guide it with `target_size_ratio` on large pools if multiple pools share the same OSD set.  
- **Multiple MDS (scale-out):** When workload grows, increase active MDS count and deploy more MDS daemons:
  ```bash
  ceph fs set cephfs max_mds 2
  ceph orch apply mds cephfs --placement="master,worker1"
  ```

---

## Optional: Quick Mount Instructions

### Create a client with CephFS caps
```bash
ceph fs authorize cephfs client.cephfs / rw
ceph auth get client.cephfs
```

Save the key to `/etc/ceph/ceph.client.cephfs.keyring`:
```
[client.cephfs]
    key = AQB...==
```

### Mount (kernel client)
```bash
mkdir -p /mnt/cephfs
mount -t ceph <mon1-ip>:6789,<mon2-ip>:6789,<mon3-ip>:6789:/ /mnt/cephfs \
  -o name=cephfs,secretfile=/etc/ceph/ceph.client.cephfs.keyring
```

### Or via ceph-fuse
```bash
mkdir -p /mnt/cephfs
ceph-fuse -n client.cephfs /mnt/cephfs
```

---

**Done.** Your CephFS `cephfs` is now backed exclusively by the `rootfs` HDD OSDs, with the autoscaler correctly hinted for bulk data.
