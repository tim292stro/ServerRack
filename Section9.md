## Section 9. Machine-to-Machine Orchestration Strategy

### 1. Passwordless Secure Connection Layer
Create an SSH deployment profile on your Linux Mint PC and distribute it to both nodes:
```bash
ssh-keygen -t ed25519 -C "mgmt-mint-pc"
ssh-copy-id root@192.168.99.2
ssh-copy-id root@192.168.99.6
```

### 2. Centralized GUI Hypervisor Control
Open **Virt-Manager** on your Linux Mint desktop workspace. Go to **File → Add Connection**. Check **"Connect to remote host over SSH"**, enter `root` as the user, and target `192.168.99.2` (Server 1). Repeat for Server 2 (`192.168.99.6`). You can now manage VMs and open graphical spice consoles cleanly from your desktop.

### 3. Configuration Syncing via Ansible Playbooks
Deploy updates using an **Ansible Playbook** from your Linux Mint terminal:

Create `~/infra-cluster/hosts.ini`:
```ini
[hypervisors]
server1 ansible_host=192.168.99.2
server2 ansible_host=192.168.99.6

[hypervisors:vars]
ansible_user=root
```

Create `~/infra-cluster/deploy-certs.yml` to automatically handle Certbot token redistribution out-of-band whenever a renewal hits:
```yaml
---
- name: Distribute Renewed SSL Certificates to Cluster
  hosts: hypervisors
  tasks:
    - name: Ensure SSL directories exist on cluster persistent storage
      ansible.builtin.file:
        path: /mnt/ha-storage/ssl-certs/
        state: directory
        mode: '0755'

    - name: Push active private keys and full chains to GlusterFS path
      ansible.posix.synchronize:
        src: "/etc/letsencrypt/live/://example.com"
        dest: "/mnt/ha-storage/ssl-certs/"
        delete: yes
        recursive: yes

    - name: Hot-Reload Target Guest Services
      ansible.builtin.shell: |
        ssh -o StrictHostKeyChecking=no root@172.16.1.100 'systemctl reload nginx'
        ssh -o StrictHostKeyChecking=no root@172.16.1.150 'kamcmd tls.reload'
      failed_when: false
```

Map this playbook to execute automatically by dropping an execution hook script onto Linux Mint at `/etc/letsencrypt/renewal-hooks/deploy/sync-to-cluster.sh`:
```bash
#!/bin/bash
cd /home/user/infra-cluster/
ansible-playbook -i hosts.ini deploy-certs.yml
```
Make the hook executable: `sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/sync-to-cluster.sh`
