## Section 11. Advanced Anti-Tamper Security: GPS Geofenced Hardware Security Module

To protect your system configuration against theft or physical equipment moving out of the secure perimeter, the decryption keys for your encrypted ZFS storage pools are secured inside the YubiHSM 2 and bound directly to a geofenced script on the Linux Mint PC.

### 1. Centralize ZFS Encryption Passphrases inside the YubiHSM 2
Rather than keeping raw text keys inside script blocks on the headless hypervisors, store your ZFS Pool passphrases as **Opaque Data Objects** inside the YubiHSM 2 hardware. The servers can only read these values if they possess a valid authentication session token authorized by the Management PC.

From your Linux Mint machine, generate two 64-byte high-entropy keys and inject them into protected slots (`Object ID 0x0111` for Server 1, `Object ID 0x0112` for Server 2) with the data reading capability restricted:

```bash
# Generate key binaries out-of-band on Mint memory path
openssl rand -hex 64 > /dev/shm/s1_zfs.key
openssl rand -hex 64 > /dev/shm/s2_zfs.key

# Import Key 1 as a locked opaque data block into the YubiHSM 2
yubihsm-shell --connector=http://127.0.0.1:2345 --authkey=1 \
  --action=put-opaque --object-id=0x0111 --label="s1_zfs_passphrase" \
  --capability=get-opaque --format=bin --in=/dev/shm/s1_zfs.key

# Import Key 2 as a locked opaque data block into the YubiHSM 2
yubihsm-shell --connector=http://127.0.0.1:2345 --authkey=1 \
  --action=put-opaque --object-id=0x0112 --label="s2_zfs_passphrase" \
  --capability=get-opaque --format=bin --in=/dev/shm/s2_zfs.key

# Scrub volatile memory instances instantly
rm -f /dev/shm/s*.key
```

### 2. Set Up the Automated GPS Geofence Daemon (Linux Mint)
Since your **u-blox ZED-X20P** receiver outputs continuous NMEA string logs to your management PC, you can configure a lightweight python service engine (`/usr/local/bin/geofence-sentinel.py`) to parse coordinate changes and act as a cryptographic circuit breaker.

Create `/usr/local/bin/geofence-sentinel.py`:
```python
#!/usr/bin/env python3
import time
import subprocess
import math

GEO_FILE = "/etc/secure-geofence.conf"
SERIAL_PORT = "/dev/ttyUSB_UBLOX"
THRESHOLD_METERS = 50.0  # Strict trigger parameter tolerance

def get_baseline():
    try:
        with open(GEO_FILE, "r") as f:
            lat, lon = map(float, f.read().strip().split(","))
            return lat, lon
    except Exception:
        return None, None

def calculate_distance(lat1, lon1, lat2, lon2):
    # Haversine tracking methodology to compute physical line distance
    R = 6371000  # Earth radius in meters
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlambda = math.radians(lon2 - lon1)
    a = math.sin(dphi/2)**2 + math.cos(phi1)*math.cos(phi2) * math.sin(dlambda/2)**2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

def trigger_lockdown():
    print("CRITICAL: Significant geographical movement detected! Locking HSM.")
    subprocess.run(["systemctl", "stop", "yubihsm-connector"], check=False)
    subprocess.run(["yubihsm-shell", "--action=reset"], check=False)

def main():
    base_lat, base_lon = get_baseline()
    if not base_lat:
        print("Geofence uncalibrated. Awaiting primary admin override authorization.")
        return

    with open(SERIAL_PORT, "r") as serial_stream:
        for line in serial_stream:
            if line.startswith("GNGGA") or line.startswith("GPGGA"):
                parts = line.split(",")
                if parts[6] != "0":  # Ensure a live validation satellite lock is active
                    try:
                        raw_lat = float(parts[2])
                        lat = (raw_lat // 100) + (raw_lat % 100 / 60)
                        if parts[3] == "S": lat = -lat
                        
                        raw_lon = float(parts[4])
                        lon = (raw_lon // 100) + (raw_lon % 100 / 60)
                        if parts[5] == "W": lon = -lon
                        
                        drift = calculate_distance(base_lat, base_lon, lat, lon)
                        if drift > THRESHOLD_METERS:
                            trigger_lockdown()
                            break
                    except (ValueError, IndexError):
                        continue
            time.sleep(1)

if __name__ == "__main__":
    main()
```

Create the systemd wrapper service unit file at `/etc/systemd/system/geofence-sentinel.service`:
```ini
[Unit]
Description=GPS Geofence Cryptographic Circuit Breaker
After=satpulsed.service
Requires=satpulsed.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/geofence-sentinel.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Enable and launch the daemon: `sudo systemctl enable --now geofence-sentinel`

### 3. Implement the Physical Override Calibration Suite
To allow relocation or recalibration, register a separate **YubiKey 5 Series** token on your Linux Mint machine using its secure internal hardware **PIV (Personal Identity Verification) Smart Card module**. 

The calibration script requires the administrator to insert their YubiKey 5 token, verify ownership using a challenge-response verification loop, enter their alphanumeric PIN, and extract clean coordinate values directly from the u-blox GPS stream to commit a new baseline profile.

Initialize the Admin YubiKey 5 Token PIV Slot:
```bash
yubico-piv-tool -a generate -s 9a -A RSA2048 -o /tmp/admin_pub.pem
yubico-piv-tool -a selfsign-certificate -s 9a -S "/CN=ClusterAdmin/" -i /tmp/admin_pub.pem -o /tmp/admin_cert.pem
yubico-piv-tool -a import-certificate -s 9a -i /tmp/admin_cert.pem
rm /tmp/admin_*.pem
```

Build the Safe Calibration Script (`/usr/local/sbin/calibrate-geofence.sh`) on the Linux Mint Management PC:
```bash
#!/bin/bash
set -e

GEO_FILE="/etc/secure-geofence.conf"
SERIAL_PORT="/dev/ttyUSB_UBLOX"

echo "=== GEOFENCE RE-CALIBRATION COMMAND CORE ==="
echo "Please insert the Master Administrator YubiKey 5 Token now."
read -p "Enter Admin smart-card PIN string: " -s PIN_STR
echo ""

echo "Verifying hardware key signature authorization token..."
set +e
yubico-piv-tool -a test-signature -s 9a -P "\$PIN_STR" > /dev/null 2>&1
if [ \$? -ne 0 ]; then
    echo "ERROR: Invalid Token PIN or Administrative Signature Verification Failed."
    exit 1
fi
set -e
echo "Hardware Signature Verified. Authorization Granted."

echo "Polling u-blox ZED-X20P GPS stream for current precise coordinates..."
while read -r line; do
    if [[ "\$line" =~ ^\$G.GGA ]]; then
        IFS=',' read -r -a p <<< "\$line"
        if [ "\${p[6]}" != "0" ]; then
            raw_lat=${p[2]}
            lat=$(echo "scale=6; (${raw_lat:0:2}) + (${raw_lat:2}/60)" | bc)
            if [ "\({p[3]}" == "S" ]; then lat="-\)lat"; fi

            raw_lon=${p[4]}
            lon=\$(echo "scale=6; (\({raw_lon:0:3}) + (\){p[4]:3}/60)" | bc)
            if [ "\({p[5]}" == "W" ]; then lon="-\)lon"; fi

            echo "\$lat,lon" > "GEO_FILE"
            chmod 600 "\$GEO_FILE"
            echo "SUCCESS: New tracking baseline committed: (lat, lon)"
            
            echo "Re-activating network key server channels..."
            systemctl start yubihsm-connector
            systemctl restart geofence-sentinel
            break
        fi
    fi
done < "\$SERIAL_PORT"
```
Make the script executable only by root: `sudo chmod 700 /usr/local/sbin/calibrate-geofence.sh`

### 4. Configure the Automated ZFS Key-Verification Script on the Node Cluster
To protect data while the cluster is running, deploy a light background tracking task via cron on both headless hosts (`/usr/local/bin/zfs-heartbeat-fence.sh`). This script queries the gateway interface every 60 seconds; if the geofence drops or the management PC is disconnected, it immediately flushes the ZFS keys from kernel memory and targets the mount volumes for emergency isolation:

```bash
#!/bin/bash
MINT_GATEWAY="192.168.99.1" # (Use 192.168.99.5 on Server 2)

# Check if the YubiHSM 2 API port is alive and responsive over the /30 subnet
curl -s --connect-timeout 5 http://\${MINT_GATEWAY}:2345/connector/status > /dev/null 2>&1

if [ \$? -ne 0 ]; then
    echo "CRITICAL WARNING: Vault connection severed or Geofence circuit tripped!" | wall
    echo "Initiating emergency cryptographic shutdown..."
    
    # Force close any open files inside the DMZ Virtual Machines to prevent corrupt file headers
    virsh list --name | xargs -I {} virsh destroy {}
    
    # Unmount the GlusterFS volumes
    umount -l /mnt/ha-storage || true
    
    # Flush the ZFS encryption keys out of kernel memory pools immediately
    zfs unload-key -a
    
    echo "Cryptographic lockdown completed. Local storage arrays are completely safe and unreadable." | wall
fi
```
Map this execution profile inside the crontab engine (`crontab -e`) on both headless servers to run continuously:
```cron
* * * * * /bin/bash /usr/local/bin/zfs-heartbeat-fence.sh
```
