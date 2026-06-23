### Section 13. Real-Time Security Telemetry Cluster Alerts

#### 1. Deploy the Non-Blocking Multithreaded Telemetry Listener (`/usr/local/bin/security_alert_daemon.py`)
```python
import redis
import requests
import smtplib
import threading
import sys
import time
from email.mime.text import MIMEText

# Bind exclusively to the point-to-point management interfaces
REDIS_HOST = '192.168.99.1'
REDIS_PORT = 6379
STREAM_NAME = 'emergency_stream'
CONSUMER_GROUP = 'security_group'
CONSUMER_NAME = 'mint_management_pc'

def initialize_redis_stream(client):
    try:
        # Create a persistent consumer group if it does not already exist
        client.xgroup_create(STREAM_NAME, CONSUMER_GROUP, id='0', mkstream=True)
    except redis.exceptions.ResponseError:
        pass # Group already initialized

def process_alert_worker(message_id, extension, timestamp):
    """Executes out-of-band network alerting inside an isolated background thread"""
    msg_body = f"CRITICAL ACCIDENT NOTICE: Extension {extension} dialed 911 at Unix Epoch {timestamp}."
    print(msg_body)
    
    # 1. Dispatch Asynchronous Network Webhook
    try:
        requests.post("https://example.com", json={
            "event": "911_DIAL", 
            "extension": extension, 
            "timestamp": timestamp
        }, timeout=5)
    except Exception as e:
        print(f"Telemetry Webhook Transmission Deficit: {e}", file=sys.stderr)
        
    # 2. Dispatch Local Corporate Mail routing parameters
    try:
        msg = MIMEText(msg_body)
        msg['Subject'] = f"EMERGENCY ALERT: 911 Executed by Ext {extension}"
        msg['From'] = "asterisk-cluster@example.com"
        msg['To'] = "security-response@example.com"
        with smtplib.SMTP('localhost') as server:
            server.send_message(msg)
    except Exception as e:
        print(f"Local SMTP Communication Deficit: {e}", file=sys.stderr)

    # 3. Explicitly acknowledge message receipt to clear it from the Redis stream queue
    try:
        r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
        r.xack(STREAM_NAME, CONSUMER_GROUP, message_id)
    except Exception as e:
        print(f"Redis Stream Acknowledgment Deficit: {e}", file=sys.stderr)

def main():
    print("Stratum 1 Unified On-Site Security Telemetry Listener Initialized...")
    while True:
        try:
            client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
            initialize_redis_stream(client)
            
            while True:
                # Read new unacknowledged items from the persistent transaction log stream
                messages = client.xreadgroup(CONSUMER_GROUP, CONSUMER_NAME, {STREAM_NAME: '>'}, count=1, block=5000)
                if not messages:
                    continue
                    
                for stream, payload in messages:
                    for msg_id, data in payload:
                        ext = data.get('extension')
                        ts = data.get('timestamp')
                        
                        if ext and ts:
                            # Spawn an isolated runtime worker thread to prevent execution loop blocking
                            t = threading.Thread(target=process_alert_worker, args=(msg_id, ext, ts))
                            t.daemon = True
                            t.start()
        except redis.exceptions.ConnectionError:
            print("Warning: Connection dropped. Re-establishing link to node vault interface...", file=sys.stderr)
            time.sleep(5)

if __name__ == "__main__":
    main()
```

#### 2. Manage the Daemon Execution via Systemd Unit Controller
To ensure this critical script starts automatically when your Linux Mint workstation boots, map it as a local core background daemon.

Create and write `/etc/systemd/system/security-alert.service`:
```ini
[Unit]
Description=Asynchronous High-Availability Telemetry Security Alert Daemon
After=network.target chrony.service satpulsed.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /usr/local/bin/security_alert_daemon.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Enable and launch the service tool on your Linux Mint PC:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now security-alert
```
