# B5 — Linux Server Implementation Runbook (report Chapter 5)
Hosts: **emr01** (E: reference datastore), **pacs01** (D: reference datastore), **portal01** (E: reference datastore), **siem01** (E: reference datastore), **lin-client** (D: reference datastore). Record the actual datastore path on the final laptop. All **Ubuntu Server 24.04 LTS** — single distro, UFW + AppArmor + Fail2ban (design fix #5). Ansible control node = siem01.

VMs first (Deliverable A sizing): emr01 2vCPU/2 GB/16 GB/**VMnet17** · pacs01 1vCPU/1 GB/12 GB/**VMnet13** · portal01 1vCPU/768 MB/8 GB/**VMnet14** · siem01 **2 vCPU/4 GB**/24 GB/**VMnet16** · lin-client 1vCPU/512 MB/8 GB (minimal install, single NIC — re-homed per scenario: VLAN 30=VMnet13, VLAN 50=VMnet15, VLAN 60=VMnet16).
ISO: `D:\NileCare\iso\ubuntu-24.04.3-live-server-amd64.iso` — **already downloaded and verified** (SHA256 `c3514bf0…744b` matches `https://releases.ubuntu.com/24.04.3/SHA256SUMS`, size 3,303,444,480 bytes; no download/extract step needed).

---

## 1. Base installation (§5.1) — repeat per host

Installer: default LVM, no swap partition (keep defaults), OpenSSH server enabled, **no snaps**. Post-install (example = emr01; adjust address per Table 3.3):

```yaml
# /etc/netplan/50-cloud-init.yaml (or new /etc/netplan/01-static.yaml; chmod 600)
network:
  version: 2
  ethernets:
    ens160:            # verify name with: ip -br link
      addresses: [10.10.70.21/24]
      routes:
        - to: default
          via: 10.10.70.1
      nameservers:
        addresses: [10.10.70.11, 10.10.70.12]   # domain segment → DCs
        search: [nilecare.local]
```

```bash
netplan apply
timedatectl set-timezone Africa/Cairo
# NTP: emr01 → dc01 ; pacs01/portal01/lin-client → own pfSense interface (10.10.30.1 / 10.10.40.1 / 10.10.50.1)
apt update && apt full-upgrade -y
apt install -y chrony vim curl unzip ufw fail2ban apparmor auditd rsyslog
echo "pool 10.10.70.11 iburst" >> /etc/chrony/sources.d/local.sources && systemctl restart chrony   # emr01 only
apt install -y needrestart && unattended-upgrades enable   # keep default 20auto-upgrades
# first Lynis baseline BEFORE hardening playbooks (Ch7 evidence):
curl -s https://get.mfp.io/? 2>/dev/null; apt install -y lynis || { cd /opt && git clone https://github.com/CISOfy/lynis && ln -s /opt/lynis/lynis /usr/local/bin/lynis; }
lynis audit system --quick | tee /root/lynis-before.txt    # record hardening index → placeholder for Ch9
```
Power state: build emr01 + siem01 first (7 GB with fw01); pacs01/portal01 next; lin-client last (512 MB, minimal install).

## 2. emr01 — OpenEMR + MariaDB (§5.2)

### 2.1 Web stack
```bash
apt install -y nginx mariadb-server php8.3-fpm php8.3-mysql php8.3-xml php8.3-mbstring \
  php8.3-gd php8.3-curl php8.3-zip php8.3-intl php8.3-soap php8.3-ldap ghostscript
mysql_secure_installation           # remove anon users, disallow remote root, remove test DB
mariadb -e "CREATE DATABASE openemr CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
mariadb -e "CREATE USER 'openemr'@'localhost' IDENTIFIED BY '<SET-IN-VAULT>'; GRANT ALL ON openemr.* TO 'openemr'@'localhost';"
```
```bash
# OpenEMR 7.x — record exact version used (honesty rule)
cd /tmp && curl -L -o openemr.tar.gz https://sourceforge.net/projects/openemr/files/OpenEMR%20Current/7.0.2/openemr-7.0.2.tar.gz/download
tar xzf openemr.tar.gz -C /var/www/html/ && mv /var/www/html/openemr-* /var/www/html/openemr
chown -R www-data:www-data /var/www/html/openemr
mkdir -p /var/www/html/openemr/sites/default/documents/{customizations,letter_templates,collected_rules} \
         /var/www/html/openemr/interface/main/calendar/modules/PostCalendar/pntemplates/compiled
chown -R www-data:www-data /var/www/html/openemr/sites/default
```
### 2.2 TLS (lab CA, self-signed)
```bash
mkdir -p /etc/nginx/ssl && cd /etc/nginx/ssl
openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout emr01.key -out emr01.crt -subj "/CN=emr01.nilecare.local/O=NileCare Lab/C=EG"
chmod 600 emr01.key
```
```nginx
# /etc/nginx/sites-available/openemr.conf
server {
    listen 80;  server_name emr01.nilecare.local;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2;  server_name emr01.nilecare.local;
    ssl_certificate     /etc/nginx/ssl/emr01.crt;
    ssl_certificate_key /etc/nginx/ssl/emr01.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    root /var/www/html/openemr;  index index.php;
    client_max_body_size 30M;
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
    location ~ /(sites/default/documents|interface/main/calendar/modules/PostCalendar/pntemplates/compiled) {
        allow all;  # OpenEMR requires write access here; FIM monitored (B7)
        try_files $uri =404;
    }
    location ~* \.(env|sql|conf)$ { deny all; }
}
```
```bash
ln -s /etc/nginx/sites-available/openemr.conf /etc/nginx/sites-enabled/ && rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```
Browser: `https://10.10.70.21` (accept self-signed) → OpenEMR setup wizard: DB host `localhost`, name/user `openemr`, create admin user `admin` (vaulted password). After install `rm /var/www/html/openemr/setup.php` per wizard instructions.
Evidence: **fig-5.1-openemr-login-https.png** (login page, padlock visible).

### 2.3 Audit logging to file (needed by Wazuh — §5.8, scenario 6)
```bash
mkdir -p /var/log/openemr && chown www-data:www-data /var/log/openemr && chmod 750 /var/log/openemr
echo '/var/log/openemr/audit.log { weekly rotate 8 compress missingok notifempty su www-data www-data }' > /etc/logrotate.d/openemr
```
OpenEMR GUI: *Admin → Logs → Logging Options*: enable **Audit Logging**, **User Tracking**; *Admin → Manage Modules? no — Logging → "Log file"*: set path `/var/log/openemr/audit.log` (OpenEMR ≥6 supports file logging alongside DB). Verify: access one record as a user → tail `/var/log/openemr/audit.log`.

### 2.4 Synthea clinical data (§5.2, design fix #4)
- Clinical records = Synthea. Generate on the **host PC** (Java 17): `java -jar synthea.jar -p 50 --exporter.csv.export true` → output/csv/*.csv.
- Import raw CSVs to a `synthea_raw` schema (provenance-transparent; report states records are synthetic):
```bash
mariadb -e "CREATE DATABASE synthea_raw CHARACTER SET utf8mb4;"
for f in patients encounters procedures medications allergies observations;
do mariadb synthea_raw -e "CREATE TABLE IF NOT EXISTS $f (line TEXT); LOAD DATA LOCAL INFILE '/root/synthea/csv/$f.csv' INTO TABLE $f FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' IGNORE 1 LINES;"; done
```
- Register **10 representative patients** in OpenEMR (UI or REST): IDs `PAT-0001…PAT-0010`, include **`PAT-VIP-001`** (VIP for scenario 6) — demographic data copied from Synthea output. This is a documented lab simplification (full HL7/FHIR ETL = future work, template 11.4).

### 2.5 MariaDB replication source (design fix #6 — RPO 15 min; completes in B6.4)
```ini
# /etc/mysql/mariadb.conf.d/60-replication.cnf
[mariadb]
server_id        = 1
log_bin          = /var/log/mysql/mariadb-bin
binlog_format    = ROW
binlog_do_db     = openemr
bind_address     = 0.0.0.0
```
```bash
systemctl restart mariadb
mariadb -e "CREATE USER 'repl'@'172.16.10.10' IDENTIFIED BY '<SET-IN-VAULT>';"
mariadb -e "GRANT REPLICATION SLAVE ON openemr.* TO 'repl'@'172.16.10.10';"
mariadb -e "SHOW BINARY LOG STATUS\G"   # record File + Position for B6.4
```
UFW/replication note: cloud-dr (172.16.10.10) connects over IPsec to emr01 (10.10.70.21) TCP 3306. Scope the emr01 UFW exception to that source and destination and keep other clients blocked.

## 3. pacs01 — Orthanc + DICOMweb (§5.3)
```bash
apt install -y orthanc orthanc-dicomweb orthanc-webviewer nginx dcmtk
cp /etc/orthanc/orthanc.json /etc/orthanc/orthanc.json.orig
```
Key `/etc/orthanc/orthanc.json` edits (JSON config):
```json
{
  "Name": "NileCare-PACS",
  "DicomPort": 4242,
  "HttpPort": 8042,
  "RemoteAET": { "MOD-SIM": true },
  "DicomModalities": { "MOD-SIM": ["MOD-SIM", "10.10.30.60", 11112] },
  "DicomAet": "NILE-PACS",
  "AuthenticationEnabled": true,
  "RegisteredUsers": { "radiology": "<SET-IN-VAULT>" },
  "DicomWeb": { "Enable": true, "StowModules": ["dicom-web"] },
  "RemoteAccessAllowed": true,
  "SslEnabled": false
}
```
TLS front via nginx (HTTPS 443 only):
```nginx
# /etc/nginx/sites-available/pacs.conf
server {
    listen 443 ssl;  server_name pacs01.nilecare.local;
    ssl_certificate /etc/nginx/ssl/pacs01.crt;  ssl_certificate_key /etc/nginx/ssl/pacs01.key;
    location / {
        proxy_pass http://127.0.0.1:8042;
        proxy_set_header Host $host;
    }
}
```
(Generate cert as in §2.2 with CN pacs01.)
```bash
ln -sf /etc/nginx/sites-available/pacs.conf /etc/nginx/sites-enabled/pacs.conf && rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx && systemctl restart orthanc
```
Sample data (public DICOM — design fix #4): download a small TCIA/Orthanc public sample set on the host PC, copy into `E:\NileCare\repo\dicom-samples\`, import:
```bash
# from lin-client re-homed to VLAN 30 (10.10.30.60, AET MOD-SIM):
storescu -aet MOD-SIM -aec NILE-PACS 10.10.30.21 104 sample-scan-001.dcm   # any imported sample .dcm
# or bulk: upload via https://pacs01.nilecare.local (web UI) — evidence of TLS login
curl -u radiology:'<SET-IN-VAULT>' -X POST https://10.10.30.21/dicom-web/studies --data-binary @sample-scan-001.dcm
```
Evidence: `orthanc-web-https.png` (extra), DICOM AET allowlist config screenshot.
**No Synthea here** — imaging provenance = public datasets (Declaration text, Deliverable A §14).

## 4. portal01 — nginx reverse proxy in DMZ (§5.4)
```nginx
# /etc/nginx/sites-available/portal.conf
server {
    listen 443 ssl;  server_name portal.nilecare.local;
    ssl_certificate /etc/nginx/ssl/portal.crt;  ssl_certificate_key /etc/nginx/ssl/portal.key;
    add_header X-Frame-Options DENY;  add_header X-Content-Type-Options nosniff;
    # Patients only hit the portal module of OpenEMR:
    location / {
        proxy_pass https://10.10.70.21;              # pfSense: DMZ→EMR01 443 only (B0 §5)
        proxy_set_header Host emr01.nilecare.local;
        proxy_ssl_verify off;                        # lab self-signed; production: real CA
    }
    location /interface { deny all; }                # staff-only paths blocked at the proxy
    location /adminer { deny all; }
}
```
```bash
mkdir -p /etc/nginx/ssl && openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/portal.key -out /etc/nginx/ssl/portal.crt -subj "/CN=portal.nilecare.local/O=NileCare/C=EG"
ln -sf /etc/nginx/sites-available/portal.conf /etc/nginx/sites-enabled/portal.conf && rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```
Internet path test (after B0 §4.6 port-forward): from host browser `https://<home-public-ip>` (or `https://pfSense-WAN-IP`) → portal (fig. support for §5.4; WAN test optional per B0 §7 note).

## 5. emr01 — Active Directory integration via SSSD (§5.5)
```bash
apt install -y realmd sssd sssd-tools krb5-user libnss-sss libpam-sss adcli libpam-sss sudo
realm discover nilecare.local            # must list dc01/dc02 (VLAN 70 → DCs, same subnet)
realm join nilecare.local -U administrator
realm permit -g GRP-IT-Admins realm-user  # only IT admins may log in
```
```ini
# /etc/sssd/sssd.conf (chmod 600, root)
[sssd]
domains = nilecare.local
config_file_version = 2
services = nss, pam

[domain/nilecare.local]
ad_domain = nilecare.local
krb5_realm = NILECARE.LOCAL
cache_credentials = True
default_shell = /bin/bash
fallback_homedir = /home/%u@%d
access_provider = ad
ad_access_filter = (memberOf=CN=GRP-IT-Admins,OU=Groups,OU=NileCare,DC=nilecare,DC=local)
ldap_user_extra_attrs = email:mail
use_fully_qualified_names = False
override_homedir = /home/%u
```
```bash
systemctl enable --now sssd && sss_cache -E
# sudo rule for IT admins:
echo '%GRP-IT-Admins ALL=(ALL) ALL' > /etc/sudoers.d/grp-it-admins && chmod 440 /etc/sudoers.d/grp-it-admins
id adm.it1            # → uid=… (domain) evidence
```
Evidence: **fig-5.2-domain-user-login-linux.png** — SSH session from host (10.10.60.100) to emr01 as `adm.it1` + `sudo -v` output; plus `realm list` output.

## 6. Samba share for radiology reports (§5.6, on pacs01)
```bash
apt install -y samba
smbpasswd -a radiology
cat >> /etc/samba/smb.conf <<'EOF'
[radiology-reports]
  path = /srv/samba/radiology
  valid users = @radiology
  read only = no
  create mask = 0660
EOF
mkdir -p /srv/samba/radiology && chown radiology:radiology /srv/samba/radiology
systemctl enable --now smbd
```
Access from Clinical (VLAN 10) is **blocked by pfSense** by design (reports live on dc02 shares; pacs01 SMB is intra-VLAN 30 only — document in §5.6 text).

## 7. Ansible automation (§5.7) — control node on siem01

Team handoff: Shaker supplies the base siem01 VM for Gate 3. Youssef owns the Ansible controller configuration and runs the Linux automation. Shaker leads integrated Wazuh enrollment at Gate 5 with Youssef's Linux-side support (§7.4).
```bash
# on siem01
apt install -y ansible-core sshpass python3-pip
mkdir -p /opt/nilecare/{playbooks,group_vars,host_vars,files} && cd /opt/nilecare
```
```ini
# /opt/nilecare/ansible.cfg
[defaults]
inventory = ./inventory.ini
host_key_checking = False
roles_path = ./roles
stdout_callback = yaml
# /opt/nilecare/inventory.ini
[linux_servers]
emr01  ansible_host=10.10.70.21
pacs01 ansible_host=10.10.30.21
portal01 ansible_host=10.10.40.10
siem01 ansible_host=10.10.60.21 ansible_connection=local
[linux_clients]
lin-client ansible_host=<per-rehoming table>
[linux:children]
linux_servers
linux_clients
[windows]
dc01      ansible_host=10.10.70.11
dc02      ansible_host=10.10.70.12
win-client ansible_host=10.10.10.110
[windows:vars]
ansible_connection=winrm
ansible_port=5985
ansible_user=adm.it1@nilecare.local
ansible_winrm_transport=ntlm
# ansible_password: NEVER in this file — ansible-vault create group_vars/windows/vault.yml (see B-README "Secrets & repo hygiene")
[linux:vars]
ansible_user=opsadmin
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```
Windows WinRM prep (run once per Windows host as admin — from B4 hosts):
```powershell
# ConfigureRemotingForAnsible.ps1 (Ansible repo script) equivalents:
winrm quickconfig -force
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Service\Auth\Basic $true
Set-Item WSMan:\localhost\Service\Auth\AllowUnencrypted $true   # lab-only: plaintext segment is internal; production = 5986/TLS (documented in report)
```
Linux prep: `useradd -m -s /bin/bash opsadmin && usermod -aG sudo opsadmin`, push `~/.ssh/id_ed25519.pub` (ansible `authorized_key`), then `ansible all -m ping`.

### 7.1 users.yml
```yaml
- name: Manage local admin account
  hosts: linux:!siem01
  become: true
  tasks:
    - name: opsadmin user
      ansible.builtin.user:
        name: opsadmin
        shell: /bin/bash
        groups: sudo
        state: present
    - name: Authorized key
      ansible.posix.authorized_key:
        user: opsadmin
        key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    - name: Password auth off (keys only)
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: restart sshd
  handlers:
    - name: restart sshd
      ansible.builtin.service: { name: ssh, state: restarted }
```
### 7.2 hardening.yml
```yaml
- name: Baseline hardening (Ubuntu 24.04 — UFW/AppArmor/Fail2ban)
  hosts: linux
  become: true
  tasks:
    - name: sysctl hardening
      ansible.posix.sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        sysctl_set: true
        reload: true
      loop:
        - { key: net.ipv4.conf.all.rp_filter,        value: "1" }
        - { key: net.ipv4.conf.all.accept_redirects, value: "0" }
        - { key: net.ipv4.conf.all.send_redirects,   value: "0" }
        - { key: kernel.dmesg_restrict,              value: "1" }
        - { key: net.ipv4.tcp_syncookies,            value: "1" }
    - name: sshd hardening
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-hardening.conf
        content: |
          PermitRootLogin no
          MaxAuthTries 4
          X11Forwarding no
          ClientAliveInterval 300
          AllowGroups sudo GRP-IT-Admins
      notify: restart sshd
    - name: UFW default deny incoming
      community.general.ufw: { direction: incoming, policy: deny }
    - name: UFW allow SSH from MGMT only
      community.general.ufw:
        rule: allow
        src: 10.10.60.0/24
        port: 22
        proto: tcp
    - name: UFW allow service ports (host-specific — from host_vars/<host>.yml)
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: "{{ ufw_service_ports | default([]) }}"
    - name: UFW enable
      community.general.ufw: { state: enabled }
    - name: fail2ban jail (ssh + nginx where present)
      ansible.builtin.copy:
        dest: /etc/fail2ban/jail.local
        content: |
          [sshd]
          enabled  = true
          maxretry = 5
          findtime = 600
          bantime  = 3600
          backend  = systemd
          [nginx-limit-req]
          enabled  = {{ 'true' if 'nginx' in group_names else 'false' }}
      notify: restart fail2ban
    - name: AppArmor enforce (default profiles)
      ansible.builtin.command: aa-enforce /etc/apparmor.d/*
      changed_when: false
      failed_when: false
    - name: auditd baseline rules
      ansible.builtin.copy:
        dest: /etc/audit/rules.d/50-nilecare.rules
        content: |
          -w /etc/passwd -p wa -k identity
          -w /etc/group  -p wa -k identity
          -w /etc/sudoers -p wa -k sudoers
          -w /etc/ssh/sshd_config -p wa -k sshd
        mode: "0640"
      notify: restart auditd
  handlers:
    - name: restart sshd
      ansible.builtin.service: { name: ssh, state: restarted }
    - name: restart fail2ban
      ansible.builtin.service: { name: fail2ban, state: restarted }
    - name: restart auditd
      ansible.builtin.service: { name: auditd, state: restarted }
```
(Per-host ports: set `ufw_service_ports` in `host_vars/<host>.yml`; the inline `vars` above shows intent — **use host_vars**, Ansible merges last-defined per host. Example `host_vars/emr01.yml`: `ufw_service_ports: [443]`.)
### 7.3 packages.yml
```yaml
- name: Baseline packages per role
  hosts: linux
  become: true
  tasks:
    - name: Common
      ansible.builtin.apt:
        name: [vim, curl, unzip, chrony, ufw, fail2ban, auditd, rsyslog, lynis, unattended-upgrades]
        state: present
        update_cache: true
    - name: EMR stack (emr01)
      ansible.builtin.apt: { name: [nginx, mariadb-server, php8.3-fpm], state: present }
      when: inventory_hostname == "emr01"
    - name: PACS stack (pacs01)
      ansible.builtin.apt: { name: [orthanc, orthanc-dicomweb, orthanc-webviewer, dcmtk], state: present }
      when: inventory_hostname == "pacs01"
    - name: Modality simulator tools (lin-client)
      ansible.builtin.apt: { name: [dcmtk], state: present }
      when: inventory_hostname == "lin-client"
    - name: chrony source (per host — ntp_source set in host_vars/<host>.yml)
      ansible.builtin.copy:
        dest: /etc/chrony/sources.d/local.sources
        content: "pool {{ ntp_source }} iburst"
      notify: restart chrony
  handlers:
    - name: restart chrony
      ansible.builtin.service: { name: chrony, state: restarted }
```
Per-host `host_vars`: `ntp_source` = emr01 `10.10.70.11`; pacs01 `10.10.30.1`; portal01 `10.10.40.1`; lin-client = the pfSense interface of the **current** homing (update `host_vars/lin-client.yml` before each re-homing). Also `host_vars/emr01.yml: ufw_service_ports: [443]`, pacs01/portal01 `[443]`, siem01/lin-client `[]`.
### 7.4 wazuh-agent.yml
```yaml
- name: Deploy Wazuh agent (events → siem01:1514, enrolment → 1515)
  hosts: linux:!siem01
  become: true
  vars:
    wazuh_manager: 10.10.60.21
    # Pinned at install time; record the version in the report (honesty rule):
    wazuh_apt_repo: "https://packages.wazuh.com/4.x/apt"
    wazuh_apt_key:  "https://packages.wazuh.com/key/GPG-KEY-WAZUH"
  tasks:
    - name: GPG key
      ansible.builtin.get_url: { url: "{{ wazuh_apt_key }}", dest: /etc/apt/trusted.gpg.d/wazuh.asc, mode: "0644" }
    - name: Repo
      ansible.builtin.apt_repository:
        repo: "deb {{ wazuh_apt_repo }} stable main"
        state: present
    - name: Install agent
      ansible.builtin.apt: { name: wazuh-agent, state: present, update_cache: true }
    - name: Point agent at manager
      ansible.builtin.lineinfile:
        path: /var/ossec/etc/ossec.conf
        regexp: '<address>'
        line: "      <address>{{ wazuh_manager }}</address>"
        insertbefore: '</client>'
      notify: restart wazuh
    - name: Read OpenEMR audit log (emr01 only)
      ansible.builtin.blockinfile:
        path: /var/ossec/etc/ossec.conf
        insertafter: '<ossec_config>'
        block: |
          <localfile>
            <log_format>syslog</log_format>
            <location>/var/log/openemr/audit.log</location>
          </localfile>
      when: inventory_hostname == "emr01"
      notify: restart wazuh
    - name: Read fail2ban log
      ansible.builtin.blockinfile:
        path: /var/ossec/etc/ossec.conf
        insertafter: '<ossec_config>'
        block: |
          <localfile>
            <log_format>syslog</log_format>
            <location>/var/log/fail2ban.log</location>
          </localfile>
      notify: restart wazuh
  handlers:
    - name: restart wazuh
      ansible.builtin.service: { name: wazuh-agent, state: restarted, enabled: true }
```
Windows agent variant (Ansible `win_package`, msi from packages.wazuh.com, `/q WAZUH_MANAGER=10.10.60.21`) — run against `[windows]` after B4 §9.

```bash
cd /opt/nilecare && ansible-playbook users.yml && ansible-playbook hardening.yml \
  && ansible-playbook packages.yml && ansible-playbook wazuh-agent.yml
```
Evidence: playbook run outputs → `ansible-<name>.log` in evidence.

## 8. Central logging — rsyslog → siem01 (§5.8)
```bash
# on every Linux host (emr01, pacs01, portal01, lin-client):
echo '*.* @@10.10.60.21:514' > /etc/rsyslog.d/90-forward-siem.conf
systemctl restart rsyslog
logger -t nilecare "test syslog forwarding"   # sender-side test; verify receipt after the Gate 5 Wazuh listener is ready
```
pfSense also forwards firewall + Suricata events to `10.10.60.21:514` (B0 §5). On siem01 the Wazuh manager accepts syslog (`<remote>` in §B7.4) so firewall/IDS/OS events land in one dashboard.

## Evidence register (chapter 5)

| File | Content | Template figure |
|---|---|---|
| fig-5.1-openemr-login-https.png | OpenEMR login over TLS | Figure 5.1 |
| fig-5.2-domain-user-login-linux.png | adm.it1 SSH + sudo on emr01 | Figure 5.2 |
| realm-sssd-output.png / sssd.conf listing | Appendix A | §5.5 |
| ansible-playbooks-run.png | playbook summary | §5.7 |
| orthanc-aet-config.png | RemoteAET allowlist | §5.3 |
| lynis-before.txt (each host) | baseline index (placeholder until run) | §9.3 |

**Power policy:** pacs01/portal01 off after build (snapshot `pacs01-baseline`, `portal01-baseline`); emr01+siem01 stay on for B6–B8.
