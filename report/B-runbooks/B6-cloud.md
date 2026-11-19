# B6 — Cloud Deployment and Hybrid Connectivity Runbook (report Chapter 6)
Cloud: **Azure for Students** ($100 credit; if you have a standard Azure Free account, the always-free/12-month quotas apply — record which). Hosts: **cloud-dr** (Ubuntu 24.04, B1s 1 vCPU/1 GB) + Blob storage with immutability. On-prem endpoint: fw01.
Local tools: Azure CLI on the host PC (`winget install Microsoft.AzureCLI`).

Honest notes: (1) Azure VPN Gateway is **not free** → we use strongSwan policy IPsec on cloud-dr (documented in the report). (2) Prefer region **UAE North** (data-residency story for §6.6); if the student subscription does not offer it, use your nearest region and record it. (3) All data in Azure is synthetic.

---

## 1. Resource group, VNet, NSG (§6.1)

```bash
az login                                     # opens browser; use the student subscription
az account show -o table                     # record subscription id → evidence

# --- Region/SKU pre-flight (Azure for Students quotas differ from pay-as-you-go) ---
az provider show --namespace Microsoft.Compute -o tsv --query "registrationState"   # expect: Registered
az vm list-skus --location uaenorth --size Standard_B1s -o table                    # expect a Standard_B1s row with no restriction column set
```
**If `Standard_B1s` or `uaenorth` is unavailable to the subscription:** choose a fallback from
`az vm list-skus --location uaenorth --output table | grep Standard_B` (e.g. `Standard_B2ats_v2`, billed to the student credit), or switch region by replacing **every** `-l uaenorth` / `--location uaenorth` in this runbook (§1 group, §5 storage account) with the fallback location (e.g. `westeurope`). For report §6.6: if the effective region is outside the intended territory, state it and point to the PDPL cross-border transfer analysis in Deliverable D — the lab itself remains compliant because **all stored data is synthetic (no personal data is processed)**.

```bash
az group create --name nilecare-dr --location uaenorth

az network vnet create -g nilecare-dr -n vnet-dr \
  --address-prefix 172.16.0.0/16 --subnet-name snet-dr --subnet-prefix 172.16.10.0/24

az network nsg create -g nilecare-dr -n nsg-dr
# Management only via the VPN tunnel; IPsec from the home WAN:
az network nsg rule create -g nilecare-dr --nsg-name nsg-dr -n AllowSSH-viaVPN \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.10.60.0/24 --destination-port-ranges 22
az network nsg rule create -g nilecare-dr --nsg-name nsg-dr -n AllowIPsec \
  --priority 110 --direction Inbound --access Allow --protocol Udp \
  --source-address-prefixes '*' --destination-port-ranges 500 4500
az network nsg rule create -g nilecare-dr --nsg-name nsg-dr -n DenyAll \
  --priority 4096 --direction Inbound --access Deny --protocol '*' \
  --source-address-prefixes '*' --destination-port-ranges '*'
```
Evidence: `fig-6.1-cloud-network-diagram.png` — draw.io from these values (vnet-dr 172.16.0.0/16, snet-dr, NSG rules, cloud-dr, Blob).

## 2. cloud-dr VM (§6.2)

```bash
az vm create -g nilecare-dr -n cloud-dr \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_B1s --vnet-name vnet-dr --subnet snet-dr \
  --nsg nsg-dr --public-ip-sku Standard --private-ip-address 172.16.10.10 \
  --admin-username opsadmin --ssh-key-values ~/.ssh/id_ed25519.pub \
  --os-disk-size-gb 30
# record public IP:
az vm list-ip-addresses -g nilecare-dr -o table
```
(If `Standard_B1s` quota errors appear, use B2s with the student credit and record the size.)
Hardening via the B5 Ansible inventory (add `[dr] cloud-dr ansible_host=172.16.10.10`):
```bash
ansible cloud-dr -m ping -i inventory.ini
ansible-playbook users.yml hardening.yml packages.yml wazuh-agent.yml -l cloud-dr
```
The Wazuh agent reaches siem01 **through the tunnel** (phase-2 selector 10.10.60.0/24 ↔ 172.16.10.0/24, §4).
Swap: configured in §4 together with the MariaDB sizing (2 GB swapfile on the 1 GB SKU).

## 3. Site-to-site VPN — strongSwan on cloud-dr + IPsec on fw01 (§6.3)

### 3.1 cloud-dr side
```bash
sudo apt install -y strongswan strongswan-pki
```
```conf
# /etc/ipsec.conf
config setup
  charondebug="ike 1, knl 1, net 0"

conn nilecare-onprem
  type=tunnel
  authby=psk
  keyexchange=ikev2
  left=%defaultroute
  leftid=@cloud-dr
  leftsubnet=172.16.10.0/24
  right=<HOME_PUBLIC_IP>          # pfSense WAN upstream (VMnet8 NAT) — record it; update if it changes
  rightid=@fw01
  rightsubnet=10.10.70.0/24,10.10.60.0/24
  ike=aes256-sha256-modp2048!
  esp=aes256-sha256-modp2048!
  ikelifetime=8h
  lifetime=1h
  dpdaction=restart
  auto=start
```
```
# /etc/ipsec.secrets (chmod 600) — PSK stored in the team vault, never in the repo
@cloud-dr @fw01 : PSK "<SET-IN-VAULT>"
```
```bash
sudo ipsec restart && sudo ipsec statusall
```
### 3.2 pfSense side — `VPN → IPsec`
- **Phase 1**: IKEv2, AES-256-GCM/SHA-256, DH 14, key lifetime 8 h; Remote Gateway = cloud-dr **public IP**; PSK as above; Local identifier `fw01` / Peer `cloud-dr`; NAT traversal on; DPD 10 s.
- **Phase 2** ×2: `10.10.70.0/24 ↔ 172.16.10.0/24` (ESP AES-256-SHA256, PFS DH14, lifetime 1 h) and `10.10.60.0/24 ↔ 172.16.10.0/24`.
- IPsec rules: pass TCP from cloud-dr (172.16.10.10) to emr01 (10.10.70.21) port 3306 only for MariaDB replication. Permit cloud-dr Wazuh event delivery to siem01 TCP 1514 as listed in A. Restrict the matching emr01 UFW rule to cloud-dr. Clinical/Admin DR access uses the cutover rules in B8 §4.
- Dynamic home IP honesty note: pfSense initiates outbound to the static Azure IP, so inbound source-NAT does not matter for IKE; if the tunnel drops after a home-IP change, update the pfSense Phase-1 remote gateway.
- Home-router caveat (same as B0 §7): the VMnet8 NAT hides pfSense behind the host; IPsec works **outbound-initiated** from pfSense without router changes in most consumer setups because ESP/UDP4500 state is created by pfSense; if the router blocks ESP, enable "Disable ESP" (UDP encapsulation) on the pfSense Phase-1 advanced settings — tested state recorded as evidence.

### 3.3 Tunnel validation
```bash
# cloud-dr:
sudo ipsec statusall | grep nilecare-onprem     # INSTALLED state
ping -c3 10.10.70.21
# pfSense: Status → IPsec → "Connected" (screenshot = fig-6.2)
```
Evidence: **fig-6.2-vpn-tunnel-status.png** (both ends).

## 4. MariaDB async replica on cloud-dr (§6.2 + design fix #6)
```ini
# /etc/mysql/mariadb.conf.d/60-replica.cnf (cloud-dr) — sized for B1s (1 GB RAM + 2 GB swap)
[mariadb]
server_id = 2
relay_log = /var/log/mysql/relay-bin
log_bin   = /var/log/mysql/cloud-bin
binlog_format = ROW
bind_address = 127.0.0.1
innodb_buffer_pool_size = 192M
innodb_log_file_size    = 32M
key_buffer_size         = 16M
tmp_table_size          = 16M
max_connections         = 30
performance_schema      = OFF
skip-name-resolve
```
```bash
# Swap first (B1s has 1 GB RAM):
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
grep -q swapfile /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo systemctl restart mariadb
sudo mariadb -e "CREATE DATABASE openemr CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
sudo mariadb -e "STOP REPLICA; RESET REPLICA ALL;"
sudo mariadb -e "CHANGE REPLICATION SOURCE TO SOURCE_HOST='10.10.70.21', SOURCE_USER='repl', SOURCE_PASSWORD='<SET-IN-VAULT>', SOURCE_LOG_FILE='<File from B5 §2.5>', SOURCE_LOG_POS=<Position>, SOURCE_CONNECT_RETRY=30;"
sudo mariadb -e "START REPLICA; SHOW REPLICA STATUS\G"    # Replica_SQL_Running: Yes → evidence
```
**Health check (run every 5 min; alert threshold = 900 s):**
```bash
sudo mariadb -e "SHOW REPLICA STATUS\G" | grep -E 'Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source'
# cron: */5 * * * * mariadb -e "SHOW REPLICA STATUS\G" | grep Seconds_Behind_Source | awk '{print $2}' >> /var/log/replica-lag.log
```
**Replication broke — recovery procedure** (referenced from B8 §1.1):
```bash
# 1) Read the failure: SHOW REPLICA STATUS\G → Last_IO_Error / Last_SQL_Error
#    - IO error, tunnel down          → fix VPN first (sudo ipsec statusall; pfSense Phase-1) — replication resumes by itself
#    - auth failed                    → re-grant on emr01: GRANT REPLICATION SLAVE ON openemr.* TO 'repl'@'172.16.10.10';
#    - binlog purged on source        → full resync (step 2)
# 2) Full resync:
#    emr01:    mariadb-dump --single-transaction --source-data --databases openemr | ssh cloud-dr 'mariadb openemr'
#    cloud-dr: STOP REPLICA; RESET REPLICA ALL;
#              CHANGE REPLICATION SOURCE TO SOURCE_HOST='10.10.70.21', SOURCE_USER='repl',
#                SOURCE_PASSWORD='<SET-IN-VAULT>', SOURCE_LOG_FILE='<File shown by the dump>', SOURCE_LOG_POS=<Position>;
#              START REPLICA; SHOW REPLICA STATUS\G
# 3) Confirm Seconds_Behind_Source → 0 and re-run the wire test from B8 §1.1.
```
RPO check (recorded for scenario 7/8): insert a test row on emr01, verify it appears on cloud-dr within seconds; measure lag with `SHOW REPLICA STATUS` `Seconds_Behind_Source`.

## 5. Immutable backup storage — Blob (§6.4)
```bash
# Immutable storage (with versioning) must be enabled AT ACCOUNT CREATION.
# Storage account names are globally unique → generate a random suffix (valid shell, no placeholder brackets):
SUFFIX="$RANDOM$RANDOM"
STG="nilecarebkup$SUFFIX"
echo "storage account = $STG"        # record in evidence; the name lives only in this shell session

az storage account create -g nilecare-dr -n "$STG" -l uaenorth \
  --sku Standard_LRS --min-tls-version TLS1_2 --https-only true \
  --enable-versioning true --immutable-storage-with-versioning true

export AZURE_STORAGE_ACCOUNT="$STG"
export AZURE_STORAGE_KEY=$(az storage account keys list -g nilecare-dr -n "$STG" --query '[0].value' -o tsv)
az storage container create --account-name "$STG" --name backups \
  --auth-mode key --public-access off

# Immutability (WORM) — 365-day time-based retention, starts UNLOCKED (can be locked after test):
az storage container immutability-policy create \
  --account-name "$STG" --container-name backups \
  --period 365 --resource-group nilecare-dr
# Lifecycle (housekeeping of non-protected prefixes):
az storage account management-policy create -g nilecare-dr --account-name "$STG" --policy @lifecycle.json
```
Secrets note: the storage key is held **only in the shell environment** — never echo it, never paste it into runbooks, evidence files, or the repo (B-README "Secrets & repo hygiene"). For persistent use by backup hosts, write it into `/etc/nilecare/backup.env` on each backup host (chmod 600, outside the repo — template in B8 §1.2).
`lifecycle.json` (delete `scratch` prefix blobs after 30 d; immutable ones untouched):
```json
{ "rules": [ { "name": "cleanup", "enabled": true, "type": "Lifecycle",
  "definition": { "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["backups/scratch"] },
  "actions": { "baseBlob": { "delete": { "daysAfterModificationGreaterThan": 30 } } } } } ] }
```
restic repositories (one per host) — installed with the B8 timer on emr01/dc02/pacs01:
```bash
export RESTIC_REPOSITORY="azure:backups:/emr01"
export AZURE_STORAGE_ACCOUNT_NAME=$AZURE_STORAGE_ACCOUNT
export AZURE_STORAGE_ACCOUNT_KEY=$AZURE_STORAGE_KEY          # from the session above — not committed anywhere
export RESTIC_PASSWORD="<SET-IN-VAULT>"                      # production runs source /etc/nilecare/backup.env (B8 §1.2)
restic init --repository-url "$RESTIC_REPOSITORY"
```
Evidence: portal screenshot of the container's immutability policy → `blob-immutability-policy.png`.

## 6. Cloud monitoring and alerts (§6.5)
```bash
az monitor metrics alert create -g nilecare-dr -n cloud-dr-cpu \
  --scopes $(az vm show -g nilecare-dr -n cloud-dr --query id -o tsv) \
  --condition "avg Percentage CPU > 85" --window-size 5m --evaluation-frequency 1m \
  --action $(az monitor action-group create -g nilecare-dr -n ag-email --action email admin <team-email> --query id -o tsv)
az monitor metrics alert create -g nilecare-dr -n blob-capacity \
  --scopes $(az storage account show -g nilecare-dr -n $AZURE_STORAGE_ACCOUNT --query id -o tsv) \
  --condition "avg BlobCapacity > 4000000000" --window-size 6h --evaluation-frequency 1h \
  --action $(az monitor action-group show -g nilecare-dr -n ag-email --query id -o tsv)
# Cost control (student credit): use Portal → Cost Management → Budgets (the az budget CLI is deprecated for student subscriptions)
#   Budget: 10 USD/month, alert at 50/80/100% → team email.
```
Evidence: `azure-monitor-alerts.png`.

## 7. Data residency and PDPL (§6.6 — full discussion in Deliverable D)
Record for the report: region chosen, storage redundancy (LRS), the statement that **all content is synthetic**, and the pointer to Egypt PDPL No. 151/2020 analysis in Chapter 10 policies (cross-border transfer discussion — Deliverable D §4).

## Evidence register (chapter 6)

| File | Content | Template figure |
|---|---|---|
| fig-6.1-cloud-network-diagram.png | VNet/subnet/NSG/cloud-dr/Blob diagram | Figure 6.1 |
| fig-6.2-vpn-tunnel-status.png | `ipsec statusall` + pfSense IPsec status | Figure 6.2 |
| replica-status.png | `SHOW REPLICA STATUS\G` Yes/0-lag | §6.2 |
| blob-immutability-policy.png | WORM policy | §6.4 |
| azure-monitor-alerts.png | metric alerts | §6.5 |

**Cost honesty:** B1s runs ~$8–10/month from the student credit; power it off (az vm deallocate) when no DR/backup lab is scheduled (`az vm deallocate -g nilecare-dr -n cloud-dr`) and record budget spend weekly.
