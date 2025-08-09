# CephFS Client Creation & Mount Guide

## 1. Prerequisites
- A running Ceph cluster with a CephFS created (example: `datafs`).
- The client machine is **not** a MON or OSD host.
- `ceph-fuse` or kernel client installed on the client machine (`ceph-common` package).
- Network access from the client to all MONs, MDSs, and OSDs:
  - **MON ports:** 3300 (msgr2) and/or 6789 (msgr1)
  - **MDS/OSD ports:** 6800–7300 TCP

---

## 2. Create a Subvolume in CephFS (on Admin Node)
```bash
# Create a subvolume in filesystem "datafs" named "test"
ceph fs subvolume create datafs test

# Get its internal path
ceph fs subvolume getpath datafs test
# Example output:
# /volumes/_nogroup/test/55bba674-ef25-4399-b9a7-72dcfb226fbc
```

---

## 3. Create the CephFS Client with FS-Scoped Caps (on Admin Node)
```bash
ceph fs authorize datafs client.kubeuser \
  /volumes/_nogroup/test/55bba674-ef25-4399-b9a7-72dcfb226fbc rwps \
  > /tmp/ceph.client.kubeuser.keyring
```

Expected caps:
```
caps mds = "allow rwps fsname=datafs path=/volumes/_nogroup/test/..."
caps mon = "allow r fsname=datafs"
caps osd = "allow rw tag cephfs data=datafs, allow rw tag cephfs metadata=datafs"
```

---

## 4. Transfer Keyring and Config to the Remote Client
On **admin node**:
```bash
scp /tmp/ceph.client.kubeuser.keyring user@remote-client:/tmp/
```

On **remote client**:
```bash
sudo mkdir -p /etc/ceph
sudo install -m 600 /tmp/ceph.client.kubeuser.keyring /etc/ceph/
```

Create `/etc/ceph/ceph.conf` on the **remote client**:
```bash
cat <<EOF | sudo tee /etc/ceph/ceph.conf
[global]
mon_host = <mon1-ip>,<mon2-ip>,<mon3-ip>
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
fsid = <your-cluster-fsid>
EOF
```
Get FSID from the **admin node**:
```bash
ceph -s | grep id:
```

---

## 5. Test Cluster Access from Remote Client
```bash
sudo ceph -n client.kubeuser \
  -k /etc/ceph/ceph.client.kubeuser.keyring \
  -m <mon1-ip>,<mon2-ip>,<mon3-ip> -s
```
Should return cluster status without permission errors.

---

## 6. Mount the Subvolume from Remote Client
### Using ceph-fuse
```bash
sudo mkdir -p /mnt/kuber
sudo ceph-fuse \
  --client_fs datafs \
  -m <mon1-ip>,<mon2-ip>,<mon3-ip> \
  -n client.kubeuser \
  -k /etc/ceph/ceph.client.kubeuser.keyring \
  -r /volumes/_nogroup/test/55bba674-ef25-4399-b9a7-72dcfb226fbc \
  /mnt/kuber
```
Verify:
```bash
ls -l /mnt/kuber
```

### Using Kernel Client (Optional)
```bash
sudo mkdir -p /mnt/kuber
SECRET=$(sudo awk -F'= ' '/key =/ {print $2}' /etc/ceph/ceph.client.kubeuser.keyring)
sudo mount -t ceph <mon1-ip>,<mon2-ip>,<mon3-ip>:/ /mnt/kuber \
  -o name=kubeuser,secret=$SECRET,mds_namespace=datafs
```

---

## 7. Unmount
```bash
sudo umount /mnt/kuber
```
For ceph-fuse:
```bash
sudo fusermount -u /mnt/kuber
```

---

## 8. Delete the Client (on Admin Node)
If you no longer need the client:
```bash
ceph auth del client.kubeuser
```
This removes the client’s key and associated caps from the cluster.

---

## 9. Troubleshooting
- **ceph-fuse hangs or fails with FS mismatch**: Ensure you specify the correct filesystem name using `--client_fs datafs`.
- **Permission denied or FS mismatch errors**: You may need to update the MON, MDS, and OSD caps for the client. On an admin node:
```bash
ceph auth caps client.kubeuser \
  mon 'allow r' \
  mds 'allow rwps fsname=datafs path=/volumes/_nogroup/test/55bba674-ef25-4399-b9a7-72dcfb226fbc' \
  osd 'allow rw tag cephfs data=datafs, allow rw tag cephfs metadata=datafs'
```
> Force kill and unmount a stuck ceph-fuse mount:
```bash
sudo pkill -f 'ceph-fuse.*mnt/kuber' || true
sudo umount -l /mnt/kuber 2>/dev/null || true
```
This widens MON caps and ensures OSD caps include metadata access.

