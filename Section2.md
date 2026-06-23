## Section 2. Linux Mint Management Machine Configuration

The management machine runs full graphical software locally to orchestrate the cluster while acting as the network boot server, hardware time master, centralized security gatekeeper, and cluster quorum tie-breaker.

### 1. Network Interface Definition
Bind Linux Mint's dedicated KVM ports to strict `/30` subnets. Ensure **no default gateways** are specified on these lines to isolate management and boot assets from the public internet.
```bash
# Configure Mint Interface 2 to service Server 1
nmcli connection add type ethernet con-name kvm-node1 ifname eth_kvm1 \
  ip4 192.168.99.1/30 ipv4.gateway "" ipv4.method manual

# Configure Mint Interface 3 to service Server 2
nmcli connection add type ethernet con-name kvm-node2 ifname eth_kvm2 \
  ip4 192.168.99.5/30 ipv4.gateway "" ipv4.method manual
```

### 2. Serial Port Hardware Locking (`udev`)
Create permanent device symlinks based on the USB-to-Serial chipset markers of your hardware timing sources to prevent device paths from switching on reboot.

Create and open `/etc/udev/rules.d/99-stratum1-clocks.rules`:
```ini
# u-blox ZED-X20P Multi-Band Receiver
SUBSYSTEM=="tty", ATTRS{idVendor}=="1546", ATTRS{idProduct}=="0506", SYMLINK+="ttyUSB_UBLOX"

# Stanford Research Systems PRS10 Rubidium Standard
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="ttyUSB_PRS10"
```
Reload the system rules engine:
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 3. SatPulse Timing Frontend Setup
Install the SatPulse package tool on your Mint desktop:
```bash
wget https://satpulse.net
sudo apt update && sudo apt install ./satpulsed_amd64.deb
```

Write the system hardware parameters configuration file to `/etc/satpulse/satpulse.toml`:
```toml
[gps]
device = "/dev/ttyUSB_UBLOX"
baudrate = 115200
protocol = "nmea"
config = true

[gps.ublox_custom]
force_stationary = true
enabled_constellations = ["GPS", "GLONASS", "GALILEO", "BEIDOU"]

[rubidium]
enabled = true
device = "/dev/ttyUSB_PRS10"
baudrate = 9600
stop_bits = 2
flow_control = "xon_xoff"
monitor_interval = 10

[pps]
device = "/dev/pps0"        # Disciplined hardware 1PPS signal coming from the PRS10
clear_edge = false
```
Enable the daemon: `sudo systemctl enable --now satpulsed`

### 4. Chrony Master Core Configuration
Install Chrony on the Mint host:
```bash
sudo apt install -y chrony
```

Update `/etc/chrony/chrony.conf` to ingest the SatPulse socket streams and restrict broadcast capability to your servers:
```ini
# Consume stabilized atomic tracking data streams from SatPulse
refclock SOCK /var/run/satpulse/chrony.sock delay 0.000 refID ATOM precision -24 lock PPS weight 100
refclock PPS /dev/pps0 lock ATOM refID PPS precision -24 prefer

offset 0.0025
maxupdateskew 2.0
corrtimeratio 2000
fallbackdrift 0

# Authorize time requests exclusively from the point-to-point server addresses
allow 192.168.99.2
allow 192.168.99.6

local stratum 1 weight 100
```
Restart the time loop: `sudo systemctl restart chrony`

### 5. Centralized ACME Certbot Setup (Constellix Integration)
To securely pull wildcard certificates using the Constellix DNS API, run these commands from your desktop shell:
```bash
sudo apt install -y certbot python3-pip
# Install the authenticated API bindings plugin natively
sudo python3 -m pip install certbot-dns-constellix
```

Obtain your **API Key** and **Secret Key** from your Constellix Management Portal under *Edit My Info*. Map them into a local access credential configuration file at `/etc/letsencrypt/constellix.ini`:
```ini
certbot-dns-constellix:dns_constellix_apikey = YOUR_CONSTELLIX_API_KEY_HERE
certbot-dns-constellix:dns_constellix_secretkey = YOUR_CONSTELLIX_SECRET_KEY_HERE
certbot-dns-constellix:dns_constellix_endpoint = https://constellix.com
```
Secure this key verification credential string tightly:
```bash
sudo chmod 600 /etc/letsencrypt/constellix.ini
```

Execute your initial verification build loop out-of-band to claim your global root and wildcard subdomains:
```bash
sudo certbot certonly \
  --authenticator certbot-dns-constellix:dns-constellix \
  --dns-constellix-credentials /etc/letsencrypt/constellix.ini \
  -d example.com \
  -d *.example.com \
  --preferred-challenges dns \
  --non-interactive \
  --agree-tos \
  -m admin@example.com
```

### 6. Authenticated Virtual Media Storage (Samba)
Deploy an isolated storage server to stream operating system blocks directly to the server nodes at boot time.
```bash
sudo apt install -y samba qemu-utils

# Create an isolated unprivileged user profile for hardware attachments
sudo useradd -M -s /usr/sbin/nologin kvmbootuser
sudo smbpasswd -a kvmbootuser

# Prepare directory structural permissions
sudo mkdir -p /srv/smb/server-disks
sudo chown -R kvmbootuser:kvmbootuser /srv/smb/server-disks
sudo chmod 700 /srv/smb/server-disks
```

Build the raw target images that act as virtual drives for the servers:
```bash
sudo qemu-img create -f raw /srv/smb/server-disks/server1-root.img 100G
sudo qemu-img create -f raw /srv/smb/server-disks/server2-root.img 100G
sudo chown kvmbootuser:kvmbootuser /srv/smb/server-disks/*.img
```

Append this block configuration profile to `/etc/samba/smb.conf` to lock down storage streaming to the point-to-point /30 management lines:
```ini
[server-disks]
   path = /srv/smb/server-disks
   browseable = no
   read only = no
   guest ok = no
   valid users = kvmbootuser
   force user = kvmbootuser
   interfaces = 192.168.99.1 192.168.99.5
   bind interfaces only = yes
```
Restart the service: `sudo systemctl restart smbd`

### 7. Deploy GUI Admin Tools, Quorum Engine Backend, and IPMI Utilities
```bash
sudo apt install -y virt-manager virt-viewer ansible corosync-qnetd ipmitool
sudo systemctl enable --now corosync-qnetd
```

Configure local terminal shortcuts on the Mint PC for programmatic hardware chassis command execution over the `/30` lines. Append these mappings to `~/.bashrc`:
```bash
# Server 1 Chassis Commands
alias s1-status="ipmitool -I lanplus -H 192.168.99.2 -U root -P 'YourIPMIPassword' power status"
alias s1-on="ipmitool -I lanplus -H 192.168.99.2 -U root -P 'YourIPMIPassword' power on"
alias s1-shutdown="ipmitool -I lanplus -H 192.168.99.2 -U root -P 'YourIPMIPassword' power soft"
alias s1-off="ipmitool -I lanplus -H 192.168.99.2 -U root -P 'YourIPMIPassword' power off"
alias s1-reset="ipmitool -I lanplus -H 192.168.99.2 -U root -P 'YourIPMIPassword' power reset"

# Server 2 Chassis Commands
alias s2-status="ipmitool -I lanplus -H 192.168.99.6 -U root -P 'YourIPMIPassword' power status"
alias s2-on="ipmitool -I lanplus -H 192.168.99.6 -U root -P 'YourIPMIPassword' power on"
alias s2-shutdown="ipmitool -I lanplus -H 192.168.99.6 -U root -P 'YourIPMIPassword' power soft"
alias s2-off="ipmitool -I lanplus -H 192.168.99.6 -U root -P 'YourIPMIPassword' power off"
alias s2-reset="ipmitool -I lanplus -H 192.168.99.6 -U root -P 'YourIPMIPassword' power reset"
```
Activate changes: `source ~/.bashrc`

### 8. Initialize the Master YubiHSM 2 Cryptographic Guard
Plug the YubiHSM 2 directly into a rear USB port on the Mint station.
```bash
sudo apt install -y yubihsm-shell yubihsm-connector opensc
```
Modify `/etc/yubihsm-connector.yaml` to stream cryptographic signatures exclusively over the point-to-point lines:
```yaml
listen: "192.168.99.1:2345,192.168.99.5:2345"
log_level: "info"
```
Fire up the backend engine: `sudo systemctl enable --now yubihsm-connector`

Log into the device interactive shell using default administrative slot 1 credentials to generate non-exportable hardware object keys:
```bash
yubihsm-shell --connector=http://127.0.0.1:2345 --authkey=1
```
Inside the prompt context execution layer, declare the storage keys:
```text
# Generate DKIM Email Signing Module (Object ID 0x0100)
yubihsm> generate asymmetric 0 0x0100 dkim_mail_key 1 sign-pkcs1c,sign-eddsa rsa2048

# Generate SIPS VoIP SIP Core TLS Key (Object ID 0x0200)
yubihsm> generate asymmetric 0 0x0200 sips_sip_key 1 sign-pkcs1c,decrypt eccp256

yubihsm> quit
```
