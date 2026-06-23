## Section 7. High-Performance RAM-Warmed VM Engine

To bypass the random IOPS performance limitations of the 18TB spinning disks, configure a pre-boot system hook that loads virtual machine files directly into server RAM at boot time.

### 1. Allocate Node Ramdisks
Create a volatile `tmpfs` volume inside `/etc/fstab` on both nodes via the Ansible setup task:
```ini
tmpfs   /mnt/vm-ramdisk   tmpfs   rw,nodev,nosuid,size=256G,mpol=interleave   0   0
```

### 2. Intercept Boot Requests via Libvirt Hooks
Write the execution hook script to `/etc/libvirt/hooks/qemu`:
```bash
#!/bin/bash
VM_NAME="\$1"
ACTION="\$2"
PHASE="\$3"

GLUSTER_MNT="/mnt/ha-storage"
RAM_MNT="/mnt/vm-ramdisk"

if [ "\(ACTION" = "prepare" ] && [ "\)PHASE" = "begin" ]; then
    if [ -f "\(GLUSTER_MNT/\)VM_NAME.qcow2" ]; then
        # Stream the image sequentially over the 200GbE link straight into local memory
        rsync -ah --inplace "\$GLUSTER_MNT/\(VM_NAME.qcow2" "\)RAM_MNT/"
    fi
fi
```
Make it executable: `chmod +x /etc/libvirt/hooks/qemu`

### 3. Register Applications into Pacemaker HA
Register your individual virtual machines as cluster resources via Pacemaker so they fail over automatically if an entire host experiences an outage:
```bash
pcs resource create VM_WebProxy ocf:heartbeat:VirtualDomain hypervisor="qemu:///system" config="/etc/libvirt/qemu/web-proxy.xml" migration_transport="ssh" op start timeout="300s"
pcs resource create VM_Mailcow ocf:heartbeat:VirtualDomain hypervisor="qemu:///system" config="/etc/libvirt/qemu/mailcow.xml" migration_transport="ssh" op start timeout="300s"
pcs resource create VM_VoIP ocf:heartbeat:VirtualDomain hypervisor="qemu:///system" config="/etc/libvirt/qemu/voip-stack.xml" migration_transport="ssh" op start timeout="300s"
```
