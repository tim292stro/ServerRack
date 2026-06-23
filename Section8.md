## Section 8. FOSS Application Stack Detailed Configuration

Every application runs inside dedicated Debian 12 Guest VMs sitting in the RAM-Backed drive paths (`/mnt/vm-ramdisk/`).

### 1. Webhosting & Content Delivery Suite (VM IP: `172.16.1.100`)
This VM runs NGINX for SSL termination, Varnish for static CDN edge caching, and Apache to execute backend worker code.

##### Varnish Caching Layer Frontend (`/etc/varnish/default.vcl`)
```vcl
vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080"; # Redirects cache-miss requests to local NGINX
}

sub vcl_recv {
    # Cache all static multimedia components cleanly
    if (req.method == "GET" && req.url ~ "\.(png|jpg|jpeg|gif|css|js|ico|webp)\$") {
        return (hash);
    }
}
```

##### NGINX SSL and Proxy Layer (`/etc/nginx/sites-available/default`)
```nginx
server {
    listen 443 ssl http2;
    server_name ://example.com;

    # Point directly to the central out-of-band TLS blocks
    ssl_certificate /mnt/persistent-gluster/ssl-certs/fullchain.pem;
    ssl_certificate_key /mnt/persistent-gluster/ssl-certs/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080; # Hands over to Varnish cache layer
    }
}

server {
    listen 8080; # Sits directly behind Varnish
    server_name ://example.com;

    location / {
        proxy_pass http://127.0.0.1:8181; # Hands processing backend tasks to Apache
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
```

##### Apache Web Server Engine Backend (`/etc/apache2/ports.conf`)
```apache
Listen 8181
```

### 2. Email & Calendaring Suite (VM IP: `172.16.1.200`)
Deploy **Mailcow: Dockerized** to integrate Postfix, Dovecot, and SOGo CalDAV groupware seamlessly.

##### Decouple Volatile Engine from Storage
Inside the VM, mount a remote path from your host's persistent `/mnt/ha-storage/mail-data/` array directly into the VM container deployment root.

`mailcow.conf` parameters:
```ini
# Define domain constraints
DOWNLOAD_TIMEOUT=20
MAILCOW_HOSTNAME=://example.com

# Direct Docker to place high-frequency state data blocks onto the persistent storage mount
vmail_volume=/mnt/persistent-gluster/vmail
```

##### Symlink Local Cert Containers Out-of-Band
```bash
rm -f data/assets/ssl/cert.pem data/assets/ssl/key.pem
ln -s /mnt/persistent-gluster/ssl-certs/fullchain.pem data/assets/ssl/cert.pem
ln -s /mnt/persistent-gluster/ssl-certs/privkey.pem data/assets/ssl/key.pem
```

Launch the deployment from the Guest shell:
```bash
docker compose up -d
```

### 3. VoIP Telephony Framework (VM IP: `172.16.1.150`)
Kamailio sits on port `5060` handling state tracking and defense, multiplexing audio streams to local Asterisk PBX nodes.

##### Kamailio Ingress Script Core (`/etc/kamailio/kamailio.cfg`)
```route
route[REGISTRAR] {
    if (!is_method("REGISTER")) return;
    # Validate client permissions and forward parameters to Asterisk array
    \$ru = "sip:127.0.0.1:5061";
    route(RELAY);
}
```

##### Kamailio Secure TLS Parameters Configuration (`/etc/kamailio/tls.cfg`)
```ini
[server:default]
method = TLSv1.2+
certificate = /mnt/persistent-gluster/ssl-certs/fullchain.pem
private_key = /mnt/persistent-gluster/ssl-certs/privkey.pem
```

##### Asterisk PBX Logic Configuration (`/etc/asterisk/sip.conf`)
```ini
[general]
bindport=5061
bindaddr=127.0.0.1

[cluster-peer]
type=friend
context=incoming-calls
host=127.0.0.1
port=5060
```

##### Asterisk Core Dialplan Interconnect Configuration (`/etc/asterisk/extensions.conf`)
```ini
[general]
static=yes
writeprotect=no
clearglobalvars=no

[incoming-calls]
; Accept incoming traffic routed by Kamailio and bridge call payloads natively
exten => _X.,1,NoOp(Incoming high-availability call stream processed via Kamailio)
same => n,Log(NOTICE, Routing trunk payload destination: \${EXTEN})
same => n,Ringing()
same => n,Dial(PJSIP/\${EXTEN}@cluster-peer,30)
same => n,Hangup()

[internal-extensions]
; Allow local DMZ phone assets to map and handshake out-of-band
exten => _1XX,1,NoOp(Internal cluster line execution)
same => n,Dial(PJSIP/\${EXTEN},20)
same => n,VoiceMail(\${EXTEN}@default)
same => n,Hangup()
```


### 4. Shared Core Network Services (Executed Natively on Hypervisor Hosts)
DHCP and Network File System parameters handle infrastructure data sharing natively across the nodes.

##### Kea DHCP Highly Available Partner Engine (`/etc/kea/kea-dhcp4.conf`)
Configure high-availability replication across both headless hosts:
```json
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "bond0" ]
    },
    "control-socket": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "hooks-libraries": [{
        "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_ha.so",
        "parameters": {
            "high-availability": [{
                "this-server-name": "server1",
                "mode": "load-balancing",
                "peers": [
                    { "name": "server1", "url": "http://10.0.0", "role": "primary" },
                    { "name": "server2", "url": "http://10.0.0", "role": "secondary" }
                ]
            }]
        }
    }]
}
}
```

##### NFS-Ganesha Clustered Shared Storage Gateway (`/etc/ganesha/ganesha.conf`)
Direct Ganesha to export your local synchronized GlusterFS blocks cleanly over the internal core networks:
```ini
EXPORT {
    Export_Id = 77;
    Path = /mnt/ha-storage;
    Pseudo = /shared-cluster-data;
    Access_Type = RW;
    FSAL {
        Name = GLUSTER;
        Volume = "HA-VM-Storage";
        Hostname = "127.0.0.1";
    }
}
```
