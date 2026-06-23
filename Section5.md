## Section 5. High-Availability Cluster Core Setup

### 1. Cluster Quorum Configuration (With Mint as QDevice Witness)
Configure Pacemaker and Corosync on both headless nodes to complete the clustering environment.
```bash
apt install -y pacemaker corosync corosync-qdevice pcs yubihsm-pkcs11 gnutls-bin
pcs host auth server1 server2
pcs cluster setup ha_cluster server1 server2

# Register the Mint machine via its point-to-point IP address as the tie-breaker
pcs cluster qdevice add net host=192.168.99.1
pcs cluster start --all
```

Configure STONITH hardware fencing parameters on the cluster so that Pacemaker can execute an out-of-band hard-reset down the `/30` power management pipelines if split-brain occurs:
```bash
pcs resource create fence_server1 fence_ipmilan ipaddr="192.168.99.2" login="root" passwd="YourIPMIPassword" lanplus=1 action=reboot op monitor interval=60s
pcs resource create fence_server2 fence_ipmilan ipaddr="192.168.99.6" login="root" passwd="YourIPMIPassword" lanplus=1 action=reboot op monitor interval=60s
```

Expose the centralized YubiHSM PKCS#11 configuration maps locally by populating `/etc/yubihsm_pkcs11.conf` on **both nodes**:
```ini
# Server 1 hooks to 192.168.99.1 / Server 2 hooks to 192.168.99.5
connector = http://192.168.99.1:2345
debug = 0
```
Append to `/etc/environment`: `YUBIHSM_PKCS11_CONF=/etc/yubihsm_pkcs11.conf`

### 2. Network Gateway Redundancy (Keepalived)
Install **Keepalived** natively on the hosts to hold a shared virtual IP address (VIP) across physical links.

`/etc/keepalived/keepalived.conf` (Server 1 example):
```ini
vrrp_instance VI_WAN {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    virtual_ipaddress {
        203.0.113.10/24  # Floating Public WAN IP
    }
}

vrrp_instance VI_LAN {
    state MASTER
    interface bond0
    virtual_router_id 52
    priority 101
    advert_int 1
    virtual_ipaddress {
        10.0.0.1/8   # Floating Internal LAN IP (Class A)
    }
}
```

##### Keepalived Cluster Health Tracker Script (`/usr/local/bin/keepalived-check.sh`)
```bash
#!/bin/bash
# Keepalived execution script to track critical services

# 1. Verify Shorewall firewall state is active
shorewall status > /dev/null 2>&1
if [ \$? -ne 0 ]; then
    echo "HEALTH CHECK FAILED: Shorewall is inactive or stalled." >&2
    exit 1
fi

# 2. Verify Libvirt hypervisor process is running
systemctl is-active libvirtd > /dev/null 2>&1
if [ \$? -ne 0 ]; then
    echo "HEALTH CHECK FAILED: Libvirtd daemon is dead." >&2
    exit 1
fi

exit 0
```
