### Section 14. Final Cluster Verification & Operational Health Checks

To ensure that your newly deployed configurations run cleanly on your workstations and headless bare-metal platforms, execute these final verification workflows from your **Linux Mint Management PC terminal**.

#### 1. Verify Management Workstation Cryptographic Slot Mapping
Ensure the management workstation can interact with your YubiHSM 2 opaque objects over the local loopback connectors and that the geofence Sentinel has not tripped the device:
```bash
yubihsm-shell --connector=http://127.0.0.1:2345 --authkey=1 --action=list-objects
```

#### 2. Verify Local Asynchronous Security Telemetry Stream Activity
Check that your background Systemd alerting service is actively listening to the persistent Redis stream without errors, and look at the recent logs to verify it is waiting on event loops:
```bash
sudo systemctl status security-alert.service
sudo journalctl -u security-alert.service -n 50 --no-pager
```

#### 3. Run a Dry-Run Test of the Centralized Certbot Pipeline
Verify that Certbot can talk via API with Constellix out-of-band and that your Ansible deployment script executes cleanly on a successful mockup validation:
```bash
sudo certbot renew --dry-run
```

#### 4. Compile and Deploy Core Node Parity Changes via Ansible
Compile your templates and push all customized Jinja2 interface metrics, Keepalived roles, and unixODBC relational database configurations to both headless servers over your isolated `/30` routes:
```bash
cd ~/infra-cluster/
ansible-playbook -i hosts.ini site.yml
```

#### 5. Verify Stratum 1 Clock Synchronization Engine Health
Verify that the u-blox ZED-X20P and SRS PRS10 Rubidium standard are processing nanosecond phase timing data cleanly, and confirm that Chrony is servicing the networks as a valid Stratum 1 reference clock:
```bash
# Check the local serial telemetry stream
satpulse-cli monitor

# Check Chrony active tracking sources
chronyc sources -v
chronyc tracking
```
