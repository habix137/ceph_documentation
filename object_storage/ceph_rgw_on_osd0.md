
# 🐘 Ceph RGW Deployment on Specific OSD (`osd.0`)

This guide shows how to deploy a Ceph RADOS Gateway (RGW) instance with its data and metadata **restricted to a specific OSD** (e.g., `osd.0`) using a **custom CRUSH rule**.

---

## 📦 Step 1: Understand the Use of RGW Pools

RGW uses specific pools to store its various components like metadata, logs, bucket indices, and object data.

| RGW Component         | Pool Name                      | Purpose                                      |
|-----------------------|--------------------------------|----------------------------------------------|
| Root directory        | `.rgw.root`                    | Stores root zone group info (global config) |
| Control plane         | `default.rgw.control`          | Internal state machine for RGW ops          |
| Metadata              | `default.rgw.meta`             | User metadata, zone info, bucket index      |
| Logging               | `default.rgw.log`              | Operation logs                               |
| Garbage Collection    | `default.rgw.gc`               | Tracks objects marked for deletion          |
| Usage tracking        | `default.rgw.usage`            | Stats about usage per user                  |
| User ID mapping       | `default.rgw.users.uid`        | Maps user IDs to metadata                   |
| Bucket index          | `default.rgw.buckets.index`    | Maps bucket names to object lists           |
| Object data           | `default.rgw.buckets.data`     | Stores actual S3 object data                |
| Data root             | `default.rgw.data.root`        | Object namespace structure                  |

These pool names are used **by default** by RGW under the default zone/realm setup.

---

## 🛠 Step 2: Create Custom CRUSH Rule for `osd.0`

Check your OSD tree:

```bash
ceph osd tree
```

Create a CRUSH rule that restricts placement to `osd.0` only (via host `master`):

```bash
ceph osd crush rule create-simple rgw-on-osd0 master default
```

Verify it:

```bash
ceph osd crush rule dump rgw-on-osd0
```

---

## 🏗 Step 3: Create RGW Pools Using the CRUSH Rule

Run this loop to create all needed pools:

```bash
for pool in default.rgw.control default.rgw.data.root default.rgw.gc default.rgw.log \
            default.rgw.meta default.rgw.usage default.rgw.users.uid \
            default.rgw.buckets.index default.rgw.buckets.data .rgw.root; do
  ceph osd pool create $pool 8 8 replicated rgw-on-osd0
done
```

Each pool is created with replication size 1 (suitable since only `osd.0` is targeted).

---

## ⚙️ Step 4: Configure and Start RGW (`spark-master`)

Edit `/etc/ceph/ceph.conf`:

```ini
[client.rgw.spark-master]
host = master
keyring = /etc/ceph/ceph.client.rgw.spark-master.keyring
rgw_frontends = civetweb port=7480
```

Generate a key:

```bash
ceph auth get-or-create client.rgw.spark-master osd 'allow rwx' mon 'allow rw' -o /etc/ceph/ceph.client.rgw.spark-master.keyring
```

Start the RGW:

```bash
systemctl start ceph-radosgw@rgw.spark-master
```

---

## ✅ Final Validation

Check that pools were created correctly:

```bash
ceph osd pool ls detail | grep rgw
```

Ensure pools are using the correct CRUSH rule:

```bash
ceph osd pool get default.rgw.buckets.data crush_rule
```

Check that all data lands on `osd.0`:

```bash
ceph pg ls-by-pool default.rgw.buckets.data
```

Test RGW access:

```bash
curl http://<master-ip>:7480
```

---

## 🧠 Notes

- If you want replication >1, you need more OSDs under the same `host master`.
- You can use `radosgw-admin` to create custom zones and manually map pools if desired, but default setup suffices here.

