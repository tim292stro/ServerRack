## Section 4. Headless Server Architecture & Storage Setup

Perform these steps via SSH from your Linux Mint desktop terminal once the servers are running their base Debian 12 environments.

### 1. Setup ZFS RAIDZ2 and Local NVMe Caching
Configure ZFS and bundle the 12 spinning SAS drives together, offloading high-frequency synchronous operations to the local NVMe drive to prevent random I/O bottlenecking.
```bash
apt install -y linux-headers-amd64 zfsutils-linux

# Group the 12 disks using an optimized 6+6 disk stripe layout to double random write IOPS
zpool create -f -o ashift=12 -O compression=lz4 -O atime=off tank \
  raidz2 /dev/disk/by-id/sas1 ... /dev/disk/by-id/sas6 \
  raidz2 /dev/disk/by-id/sas7 ... /dev/disk/by-id/sas12

# Carve out the fast NVMe partitions to serve as Write Log (SLOG) and Cache (L2ARC)
zpool add tank log nvme-vdev-partition1
zpool add tank cache nvme-vdev-partition2

# Create the brick dataset container for GlusterFS replication
zfs create tank/gluster-brick
zfs set xattr=sa tank/gluster-brick
zfs set recordsize=128k tank/gluster-brick
```

### 2. Bind GlusterFS Replication Across the 200 GbE Interconnect
Optimize the server network buffers inside `/etc/sysctl.conf` on both hosts to stream large file operations cleanly over the Mellanox ConnectX-6 cards:
```ini
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
net.core.netdev_max_backlog = 100000
```
Apply with `sysctl -p`.

Configure network parameters inside `/etc/network/interfaces` on both servers to initialize your 200 GbE links with **Jumbo Frames**:
```ini
auto eth200g
iface eth200g inet static
    address 10.200.0.1  # (Use 10.200.0.2 on Server 2)
    netmask 255.255.255.0
    mtu 9000
```

From **Server 1**, connect the distributed filesystem layers over the 200 GbE link:
```bash
apt install -y glusterfs-server
gluster peer probe 10.200.0.2

# Build the 2-node replicated storage volume over TCP
gluster volume create HA-VM-Storage replica 2 \
  10.200.0.1:/tank/gluster-brick/b1 \
  10.200.0.2:/tank/gluster-brick/b2 transport tcp

# Apply tuning patterns to stabilize VM files hosted on spinning disk arrays
gluster volume set HA-VM-Storage performance.write-behind-window-size 128MB
gluster volume set HA-VM-Storage features.shard on
gluster volume set HA-VM-Storage features.shard-block-size 64MB
gluster volume start HA-VM-Storage

# Local client mount point on both nodes
mkdir -p /mnt/ha-storage
mount -t glusterfs 127.0.0.1:/HA-VM-Storage /mnt/ha-storage
```

##### Automated GlusterFS Systemd Mount Configuration (`/etc/systemd/system/mnt-ha\\x2dstorage.mount`)
```ini
[Unit]
Description=Automated Distributed High-Availability GlusterFS Volume Mount
After=network-online.target glusterfd.service eth200g.service
Before=libvirtd.service

[Mount]
What=127.0.0.1:/HA-VM-Storage
Where=/mnt/ha-storage
Type=glusterfs
Options=defaults,_netdev,noatime,fetch-attempts=5

[Install]
WantedBy=multi-user.target
```
