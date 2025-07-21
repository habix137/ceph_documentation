# here I want to explain how we can configure ceph file system

## create volume

first we need to create volume 
```bash
ceph fs volume create aip-test --placement="ceph2 ceph3"
```
> this command show that ceph2 and ceph3 that is ceph nodes are ceph mds node and if you don't detemine these two node automaticly assign to nodes and also you should consider that one of these node is standby to see general status use this command in cephadm shell

```bash
ceph -s
```

## ceph fs pools info
for this you can use this command 
```bash
ceph fs ls
```
example result:
```
name: aip-test, metadata pool: cephfs.aip-test.meta, data pools: [cephfs.aip-test.data ]
```

## Mounting CephFS on a Remote Server

To mount a Ceph file system on a remote server, use the `mount` command with the Ceph file system type. The general syntax is:

```bash
mount -t ceph <mon_ip>:<mon_port>:/<subvolume_path> <mount_point> -o name=<client_name>,secret=<key>,fs=<file_system_name>
```

- `<mon_ip>`: IP address of a Ceph monitor node
- `<mon_port>`: Monitor port (usually 6789)
- `<subvolume_path>`: Path to the CephFS subvolume (use `/` for the root)
- `<mount_point>`: Local directory where you want to mount CephFS
- `<client_name>`: Ceph client user (e.g., `client.admin`)
- `<key>`: Secret key for the client user
- `<file_system_name>`: Name of the Ceph file system (e.g., `aip-test`)

### Example

Suppose:
- Monitor IP: `192.168.1.10`
- Monitor port: `6789`
- File system name: `aip-test`
- Mount point: `/mnt/cephfs`
- Client name: `client.admin`
- Secret key: `AQB2...yourkey...AA==`

Mount the root of the file system:

```bash
mount -t ceph 192.168.1.10:6789:/ /mnt/cephfs -o name=client.admin,secret=AQB2...yourkey...AA==,fs=aip-test
```

If you want to mount a specific subvolume (e.g., `mydata`):

```bash
mount -t ceph 192.168.1.10:6789:/mydata /mnt/cephfs -o name=client.admin,secret=AQB2...yourkey...AA==,fs=aip-test
```

> **Note:** Make sure the `ceph-common` package is installed on your remote server.

### you can use one load balancer and use it for monitor mon in your client and for mon_ip use ip of your load balancer

## Setting File Quotas on CephFS Using setfattr

You can limit the maximum number of files in a CephFS directory or subvolume by setting an extended attribute using the `setfattr` command. This is done on the client side, after mounting the CephFS file system.

### Set Maximum Number of Files for a Directory

To set a file quota (maximum number of files) on a specific directory:

```bash
setfattr -n ceph.quota.max_files -v <number_of_files> <mount_point>/<directory>
```

- `<number_of_files>`: The maximum number of files allowed in the directory.
- `<mount_point>/<directory>`: The path to the directory on your mounted CephFS.

**Example:**  
To limit `/mnt/cephfs/mydata` to 100,000 files:

```bash
setfattr -n ceph.quota.max_files -v 100000 /mnt/cephfs/mydata
```

### Set Maximum Number of Files for the Entire Volume

To set a file quota for the root of the mounted CephFS (i.e., the whole volume):

```bash
setfattr -n ceph.quota.max_files -v <number_of_files> <mount_point>
```

**Example:**  
To limit the entire file system mounted at `/mnt/cephfs` to 1,000,000 files:

```bash
setfattr -n ceph.quota.max_files -v 1000000 /mnt/cephfs
```

> **Note:**  
> - These commands must be run on the client machine where CephFS is mounted.
> - You need appropriate permissions (usually as the CephFS admin or owner of the directory).
> - The quota takes effect immediately and applies to all files within the specified directory or volume.

