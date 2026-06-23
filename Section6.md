## Section 6. Shorewall HA Configuration Files

Create these configurations in `~/infra-cluster/shorewall-configs/` on the Linux Mint Management PC to handle multi-zone routing and virtual IP bindings dynamically.

### `/etc/shorewall/zones`
```ini
# ZONE   TYPE    OPTIONS
fw       firewall
net      ipv4
dmz      ipv4
```

### `/etc/shorewall/interfaces`
```ini
# ZONE   INTERFACE   OPTIONS
net      eth0        dhcp,nosmurfs,logmartians,required
dmz      bond0       bridge,options
```

### `/etc/shorewall/policy`
```ini
# SOURCE   DEST     POLICY    LOG LEVEL
\(FW        net      ACCEPT\)FW        dmz      ACCEPT
dmz        net      ACCEPT
net        all      DROP      info
all        all      REJECT    info
```

### `/etc/shorewall/rules`
```ini
########################################################################################################################
# Shorewall Firewall Configuration File: /etc/shorewall/rules
# Architecture: Dual-Node HA Front-End Cluster (Server 1 / Server 2)
#
# Zones:
#   \$FW   - The local firewall engine host platforms
#   net   - Public Internet WAN (eth0 / Floating Public VIP: 203.0.113.10)
#   dmz   - Private Internal LAN / DMZ (bond0 / Floating LAN VIP: 10.0.0.1)
########################################################################################################################

#ACTION         SOURCE          DEST                            PROTO   DEST_PORT

?SECTION ALL
# Drop invalid packets immediately before parsing standard rules
DROP            net             all                             all

?SECTION ESTABLISHED
# Explicitly authorize established return streams across the zones
ACCEPT          all             all

?SECTION RELATED
# Explicitly authorize related management traffic strings
ACCEPT          all             all

?SECTION NEW

#=======================================================================================================================
# SECTION 1: ANTI-SPOOFING & REJECT MALICIOUS PACKETS
#=======================================================================================================================
# Drop NetBIOS broadcast noise from leaking into systems or logs
DROP            net             all                             tcp     137,138,139
DROP            net             all                             udp     137,138,139
DROP            all             net                             udp     1900      # Block SSDP noise

# Reject ident lookups cleanly to avoid connection lag
REJECT          net             all                             tcp     113

#=======================================================================================================================
# SECTION 2: ADMINISTRATIVE OUT-OF-BAND MANAGEMENT (CLASS A LAN ONLY)
#=======================================================================================================================
# Allow SSH connections into the firewalls ONLY from the secure internal Class A subnet range
SSH(ACCEPT)     dmz:10.0.0.0/8  \$FW

# Allow the headless hypervisor nodes to run raw ping checks for cluster health monitoring
Ping(ACCEPT)    dmz:10.0.0.0/8  \$FW
Ping(ACCEPT)    \$FW             dmz:10.0.0.0/8
Ping(ACCEPT)    \$FW             net

#=======================================================================================================================
# SECTION 3: HTTP / HTTPS FRONT-END REVERSE PROXY PIPELINE (VM IP: 172.16.1.100)
#=======================================================================================================================
# Route outside web traffic straight to your Varnish CDN / NGINX SSL Termination proxy instance
HTTP(ACCEPT)    net             dmz:172.16.1.100
HTTPS(ACCEPT)   net             dmz:172.16.1.100

#=======================================================================================================================
# SECTION 4: MAILCOW GROUPWARE CORE PROTOCOL PIPELINE (VM IP: 172.16.1.200)
#=======================================================================================================================
# Standard SMTP (MTA-to-MTA mail exchange routing)
DNAT            net             dmz:172.16.1.200                tcp     25

# Secure SMTPS (Implicit TLS mail routing)
DNAT            net             dmz:172.16.1.200                tcp     465

# Mail Submission (Authenticated user client mail submission)
DNAT            net             dmz:172.16.1.200                tcp     587

# Secure IMAPS (Encrypted mail folder syncing for user profiles)
DNAT            net             dmz:172.16.1.200                tcp     993

#=======================================================================================================================
# SECTION 5: KAMAILIO & ASTERISK VOIP TELEPHONY PIPELINE (VM IP: 172.16.1.150)
#=======================================================================================================================
# Kamailio SIP Registrar Ingress (Standard and Secure TLS Signaling)
DNAT            net             dmz:172.16.1.150                tcp     5060,5061
DNAT            net             dmz:172.16.1.150                udp     5060,5061

# Asterisk RTP Media Port Range Allocation (Handles UDP audio packet payload streaming)
DNAT            net             dmz:172.16.1.150                udp     10000:20000

#=======================================================================================================================
# SECTION 6: BACK-END CORE NETWORK UTILITIES (ISOLATED TO LAN ZONE)
#=======================================================================================================================
# Authorize Kea DHCP Address Allocations over the local bridges
ACCEPT          dmz             \$FW                             udp     67,68

# Authorize Chrony NTP time broad-sync lookups against the Stratum 1 Master gateway
ACCEPT          dmz             \$FW                             udp     123

# Authorize NFS / Ganesha Storage Share communication limits
ACCEPT          dmz             \$FW                             tcp     2049

# LAST LINE -- DO NOT REMOVE
```
