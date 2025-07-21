# block devices

## you can format this disk when you assign this block device


if you want to to create rbd (radoas block device) first you should create one pool for this 

```bash
ceph osd pool create rbdpool 50
```


no you can change replication of this pool

```bash
ceph osd pool set rbdpool size 3
```

## Initializing an RBD Pool

Before you can use a pool for RADOS Block Devices (RBD), you need to initialize it. This sets up the necessary metadata in the pool so that RBD images can be created and managed.

To initialize your pool (for example, `rbdpool`), run:

```bash
rbd pool init rbdpool
```

**Explanation:**

- This command prepares the specified pool (`rbdpool`) for use with RBD by creating required internal objects.
- It is a one-time operation per pool and must be done before creating any RBD images in a new pool.
- If you skip this step, you may encounter errors when trying to create or use RBD images in the pool.

> Always run `rbd pool init <pool_name>` after creating a new pool intended for RBD usage.


## ceph info
you can see information of your storage and pools with this command

```bash
ceph df
```

here is the result:

RAW STORAGE 
| CLASS | SIZE    | AVAIL    | USED    | RAW USED | %RAW USED |
|-------|---------|----------|---------|----------|-----------|
| hdd   | 180 GiB | 180 GiB  | 384 MiB | 384 MiB  | 0.21      |
| TOTAL | 180 GiB | 180 GiB  | 384 MiB | 384 MiB  | 0.21      |

POOLS TABLE
| POOL     | ID | PGS | STORED  | OBJECTS | USED     | %USED | MAX AVAIL |
|----------|----|-----|---------|---------|----------|-------|-----------|
| .mgr     | 1  | 1   | 705 KiB | 2       | 2.1 MiB  | 0     | 57 GiB    |
| rbdpool  | 2  | 50  | 0 B     | 0       | 0 B      | 0     | 57 GiB    |


as you can see I have 180 GB in my whole storage and because of I set size 3 for rbdpool max avalable size for this disk is about 60GB 

## information of block device:

```bash
rbd ls --pool <pool_name>
```
this command check if in this pool we have rbd device show us

## create block device

```bash
rbd create <block_device_name> --size <size_rbd_meg> --pool <pool_name>
```
here I put bl1 as name of this rbd and I use rbdpool for pool that this rbd assign to it

for more information of your rbd you can use this command

```bash
rbd info --image <rbd_name> --pool <pool_name>
```
## Interpreting `rbd info` Output

When you run:

```bash
rbd info --image bl1 --pool rbdpool
```

You might see output like:

```
rbd image 'bl1':
        size 5 GiB in 1280 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: fb1c9f723132
        block_name_prefix: rbd_data.fb1c9f723132
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features:
        flags:
        create_timestamp: Mon Jul 21 06:27:24 2025
        access_timestamp: Mon Jul 21 06:27:24 2025
        modify_timestamp: Mon Jul 21 06:27:24 2025
```

**Explanation of fields:**

- **size 5 GiB in 1280 objects**: The RBD image is 5 GiB, split into 1280 objects in the pool.
- **order 22 (4 MiB objects)**: Each object is 4 MiB in size (2^22 bytes).
- **snapshot_count**: Number of snapshots for this image (here, 0).
- **id**: Unique identifier for this RBD image.
- **block_name_prefix**: Prefix used for the underlying RADOS objects.
- **format**: Image format version (2 is the current standard).
- **features**: Enabled features for this image (e.g., layering, exclusive-lock).
- **op_features**: Operational features (may be empty).
- **flags**: Additional flags (may be empty).
- **create_timestamp**: When the image was created.
- **access_timestamp**: Last access time.
- **modify_timestamp**: Last modification time.

This information helps you understand the configuration, capabilities, and usage history of your RBD image.



# add rbd to your device

## Connecting to Your Ceph Cluster from a Remote Server

The easiest way to connect your remote server to your Ceph cluster is by copying the cluster configuration file (`ceph.conf`) and the appropriate keyring file (e.g., `ceph.client.admin.keyring`) to the `/etc/ceph/` directory on your remote server. Then, install the Ceph client utilities.

**Steps:**

1. **Copy configuration and keyring files:**
   - Copy `ceph.conf` from your Ceph cluster to `/etc/ceph/` on your remote server.
   - Copy your keyring file (e.g., `ceph.client.admin.keyring`) to `/etc/ceph/` on your remote server.

2. **Install Ceph client utilities:**
   ```bash
   sudo apt install ceph-common
   ```

3. **Verify connection:**
   - You can now run Ceph commands from your remote server, such as:
     ```bash
     ceph -s
     ```

> This is the simplest way to connect a remote server to your Ceph cluster for basic management and access.  
> For production or more secure environments, it is recommended to create a dedicated Ceph client user with limited permissions, but for now, this basic setup is sufficient.


## list of rbd device on your server

```bash
rbd showmapped
```


## Mapping an RBD Image to a Block Device

To use an RBD image as a block device on your server, you need to map it. This command maps the RBD image to a device node (e.g., `/dev/rbd0`):

```bash
sudo rbd map --image <block_device_name> --pool <pool_name>
```

- This command connects the RBD image named `bl1` from the pool `rbdpool` to your system as a block device (usually `/dev/rbd0`).
- After mapping, you can use standard Linux tools (like `mkfs`, `mount`, etc.) on `/dev/rbd0`.

> Note: You need appropriate permissions and the Ceph client configuration/keyring on your server to run this command.

## Unmapping an RBD Device

When you no longer need access to an RBD image as a block device, you should unmap it from your system. This will safely detach the device node (e.g., `/dev/rbd0`).

To unmap an RBD device, use:

```bash
sudo rbd unmap /dev/rbd0
```

- Replace `/dev/rbd0` with the actual device node you want to unmap.
- This command disconnects the mapped RBD image from your system, making it safe to remove or remap.

> Always ensure the device is not in use (e.g., not mounted or busy) before unmapping.

## Formatting and Using Your RBD Device on a Remote Server

After mapping your RBD image as a block device (e.g., `/dev/rbd0`), you need to format it with a filesystem before you can use it. Here’s how to do it with the XFS filesystem:

1. **Install XFS utilities (if not already installed):**

   ```bash
   sudo apt install xfsprogs -y
   ```

   - This command installs the necessary tools to create and manage XFS filesystems.

2. **Format the RBD device with XFS:**

   ```bash
   sudo mkfs.xfs /dev/rbd0
   ```

   - This command creates an XFS filesystem on the mapped RBD device (`/dev/rbd0`).
   - Replace `/dev/rbd0` with your actual device node if different.

> After formatting, you can mount `/dev/rbd0` to a directory and start using it for storage on your remote server.

## Mounting the RBD Device to a Directory

After formatting your RBD device (e.g., `/dev/rbd0`), you can mount it to a directory on your remote server. For example, to mount it to `/mnt/exd`:

1. **Create the mount point (if it doesn't exist):**

   ```bash
   sudo mkdir -p /mnt/exd
   ```

2. **Mount the RBD device:**

   ```bash
   sudo mount /dev/rbd0 /mnt/exd
   ```

   - This command mounts the RBD device to the `/mnt/exd` directory.
   - Replace `/dev/rbd0` and `/mnt/exd` with your actual device and desired mount point if different.

> Now you can use `/mnt/exd` as a regular storage directory backed by your Ceph RBD image.


## performance test

you can test performance with dd command

```bash
dd if=/dev/zero of=/mnt/exd/file.fs bs=1M Count=1024"
```

## rbd info

```bash
rbd du --image bl1 --pool rbdpool
```

## resize rbd

```bash
rbd resize --image bl1 --size 8224 --pool rbdpool
```
then on your remote server:

```bash
xfs_growfs -d /mnt/exd
```
## Using rbdmap for Automatic RBD Mapping

The `rbdmap` service allows you to automatically map RBD images at boot or on demand by configuring `/etc/ceph/rbdmap`. This is useful for persistent RBD device mapping on your client.

### 1. Configure `/etc/ceph/rbdmap`

Edit (or create) the `/etc/ceph/rbdmap` file and add a line for each image you want to map. The format is:

```
<pool>/<image> id=<client_id> keyring=/etc/ceph/ceph.client.<client_id>.keyring
```

**Example:**

```
rbdpool/bl1 id=ali keyring=/etc/ceph/ceph.client.ali.keyring
```

- `rbdpool` is your pool name.
- `bl1` is your RBD image name.
- `ali` is your Ceph client user.
- The keyring file should contain the key for your client.

### 2. Enable and Start the rbdmap Service

To have your RBD images mapped automatically at boot, enable and start the `rbdmap` service:

```bash
sudo systemctl enable rbdmap
sudo systemctl start rbdmap
```

- `enable` ensures the service starts at boot.
- `start` maps the devices immediately.

### 3. Verify Mapped Devices

After starting the service, check mapped devices with:

```bash
rbd showmapped
```

> With this setup, your specified RBD images will be automatically mapped as block devices whenever your system boots, using the credentials and configuration you provided in `/etc/ceph/rbdmap`.

## Automatically Mounting RBD Devices with /etc/fstab

To ensure your RBD device is automatically mounted at boot, you can add an entry to `/etc/fstab`. This works together with `rbdmap`, which maps the device node (e.g., `/dev/rbd0`) before the mount.

### 1. Find Your RBD Device

After configuring `rbdmap` and starting the service, check which device node your image is mapped to:

```bash
rbd showmapped
```

Example output:
```
id  pool     image  snap  device
0   rbdpool  bl1    -     /dev/rbd0
```

### 2. Add an Entry to `/etc/fstab`

Open `/etc/fstab` with your preferred editor and add a line like this:

```
/dev/rbd0    /mnt/exd    xfs    defaults,_netdev    0    0
```

- `/dev/rbd0`: The mapped RBD device.
- `/mnt/exd`: The mount point directory.
- `xfs`: The filesystem type (use the one you formatted with).
- `defaults,_netdev`: Standard options; `_netdev` ensures the device is mounted after network is available.

### 3. Test the Mount

You can test your `/etc/fstab` entry without rebooting:

```bash
sudo mount -a
```

> With this configuration, your RBD device will be automatically mapped and mounted every time your server boots.


## Preventing Concurrent Access (Deny Sharing) for an RBD Image

By default, Ceph RBD images can be mapped by multiple clients at the same time, which can lead to data corruption if the filesystem does not support clustering. To prevent this and ensure only one client can access (map) an RBD image at a time, you should enable the `exclusive-lock` feature.

### 1. Check if `exclusive-lock` is enabled

Run the following command to see the features of your RBD image:

```bash
rbd info --image <image_name> --pool <pool_name>
```

Look for `exclusive-lock` in the features list.

### 2. Enable `exclusive-lock` if not already enabled

If `exclusive-lock` is not listed, enable it with:

```bash
rbd feature enable <image_name> exclusive-lock --pool <pool_name>
```

**Example:**

```bash
rbd feature enable bl1 exclusive-lock --pool rbdpool
```

- This ensures that only one client can map and write to the RBD image at a time.
- If another client tries to map the image while it is already in use, the operation will fail.

> Enabling `exclusive-lock` is recommended for single-writer filesystems (like XFS or ext4) to prevent data corruption due to concurrent access.

## Creating and Listing Snapshots for an RBD Image

Snapshots allow you to capture the state of an RBD image at a specific point in time. This is useful for backups, testing, or recovery.

### 1. Create a Snapshot

To create a snapshot named `snapshot1` for the image `bl1` in the pool `rbdpool`, use:

```bash
rbd snap create <rbd_pool>/<rbd_name>@<snapshot_name>
```
example:
```bash
rbd snap create rbdpool/bl1@snapshot1
```

- This command creates a snapshot called `snapshot1` for the specified RBD image.
- You will see progress output like:  
  `Creating snap: 100% complete...done.`

### 2. List Snapshots

To view all snapshots for an RBD image, use:

```bash
rbd snap ls <rbd_pool_name>/<rbd_name>
```
example:
```bash
rbd snap ls rbdpool/bl1
```

Example output:

```
SNAPID  NAME       SIZE     PROTECTED  TIMESTAMP
     4  snapshot1  8.0 GiB             Mon Jul 21 11:16:41 2025
```

- `SNAPID`: Unique ID for the snapshot.
- `NAME`: Name of the snapshot.
- `SIZE`: Size of the image at the time of the snapshot.
- `PROTECTED`: Indicates if the snapshot is protected (cannot be deleted unless unprotected).
- `TIMESTAMP`: When the snapshot was created.

> Snapshots are lightweight and can be used for quick restores or cloning new images.

## Working with RBD Snapshots: Rollback and Safe Unmapping

When you want to revert an RBD image to a previous state using a snapshot, you need to ensure the device is not in use, unmap it safely, and then perform the rollback. Here’s how to do it step by step:

### 1. Check for Open Files on the Mount Point

Before unmapping an RBD device, make sure no process is using the mount point (e.g., `/mnt/exd`). You can check this with:

```bash
sudo lsof +f -- /mnt/exd
```

- This command lists all processes that have files open on `/mnt/exd`.
- Example output:
  ```
  COMMAND    PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
  bash    842712 alsafarpour  cwd    DIR  251,0        6  128 /mnt/exd
  ssh     843030 alsafarpour  cwd    DIR  251,0        6  128 /mnt/exd
  ```

### 2. Stop Processes Using the Mount Point

If there are processes using the mount point, you need to stop them. You can kill the processes by their PID:

```bash
sudo kill -9 842712 843030
```

- Replace `842712 843030` with the actual PIDs from your `lsof` output.

### 3. Unmap the RBD Device

Once no process is using the device, unmap it:

```bash
sudo rbd unmap /dev/rbd0
```

- This detaches the RBD image from your system.

### 4. Rollback to a Snapshot

Now you can safely rollback the RBD image to a previous snapshot. For example, to rollback to `snapshot2`:

```bash
rbd snap rollback rbdpool/bl1@snapshot2
```

- This command reverts the RBD image `bl1` in pool `rbdpool` to the state it was in at `snapshot2`.
- **Warning:** Rolling back will discard all changes made to the image after the snapshot was taken.

---

**Summary of Commands:**

```bash
sudo lsof +f -- /mnt/exd
sudo kill -9 <PID1> <PID2> ...
sudo rbd unmap /dev/rbd0
rbd snap rollback rbdpool/bl1@snapshot2
```

> Always ensure the device is not in use before unmapping or rolling back to a snapshot to prevent data corruption.

then again map rbd to remote server

```bash
rbd map rbdpool/bl1
```
and mount it
```bash
mount /dev/rbd0 /mnt/exd
```
and here we go we have a backup from your data in snapshot :)

## remove snap shot

you can remove snapshots with this command

```bash
rbd snap rm <pool_name>/<rbd_name>@<snap_name>
```

example:
```bash
rbd snap rm rbdpool/bl1@snapshot1
```

