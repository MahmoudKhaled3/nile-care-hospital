# B8 — Backup and Disaster Recovery Runbook (report Chapter 8)
Implements Deliverable A Table 8.1 (3-2-1) and Table 8.2 (RTO/RPO). Depends on: B6 (Blob, VPN, replica), B5 (Ansible, timers pattern).
Power states: dc02 ON for §2–§3; cloud-dr ON (deallocated otherwise); emr01/siem01 ON.

---

## 1. EMR database — two RPO layers (design fix #6)

### 1.1 Continuous: MariaDB async replication (RPO ≤ 15 min)
- Source + replica configured in B5 §2.5 and B6 §4. Monitoring:
```bash
# emr01 (source):
mariadb -e "SHOW BINARY LOG STATUS\G"
# cloud-dr (replica): cron a lag probe every 5 min, alert via Azure metric (B6 §6) when > 900 s
sudo mariadb -e "SHOW REPLICA STATUS\G" | grep -E 'Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source'
```
- **If replication breaks:** follow the "Replication broke — recovery procedure" in B6 §4 (diagnose → fix tunnel/auth → full resync), then re-run the wire check below.
- Wire check (recorded as evidence `replication-wire-test.png`): on emr01 run the timestamped insert below; the same row must appear on cloud-dr.
```sql
-- emr01:
INSERT INTO synthea_raw.patients (line) VALUES (CONCAT('rpo-check-', NOW()));
-- cloud-dr:
SELECT * FROM synthea_raw.patients WHERE line LIKE 'rpo-check-%' ORDER BY 1 DESC LIMIT 1;
```

### 1.2 Nightly: GPG-encrypted logical dump → immutable Blob
```bash
# /usr/local/sbin/emr-db-backup.sh (emr01) — chmod 750, root
#!/bin/bash
set -euo pipefail
# Secrets & endpoints live OUTSIDE the repo, chmod 600 (see B-README "Secrets & repo hygiene"):
#   /etc/nilecare/backup.env : RESTIC_REPOSITORY, AZURE_STORAGE_ACCOUNT_NAME,
#                              AZURE_STORAGE_ACCOUNT_KEY, RESTIC_PASSWORD
#   /root/.gpg-pass          : passphrase of the ops GPG key
source /etc/nilecare/backup.env
STAMP=$(date +%Y%m%d-%H%M)
OUT=/var/backups/emr
mkdir -p "$OUT"
mysqldump --single-transaction --routines --triggers --source-data \
  --databases openemr | gzip | \
  gpg --batch --yes --pinentry-mode loopback --passphrase-file /root/.gpg-pass \
      --encrypt --recipient ops@nilecare.local \
      --output "$OUT/openemr-$STAMP.sql.gz.gpg"
find "$OUT" -name 'openemr-*.gpg' -mtime +7 -delete
restic backup "$OUT" --tag emr-db >> /var/log/emr-backup.log 2>&1
restic forget --keep-daily 30 --prune >> /var/log/emr-backup.log 2>&1
logger -t emr-backup "dump+restic completed $STAMP"
```
```bash
# /etc/nilecare/backup.env — TEMPLATE (fill from the B6 session; chmod 600; NEVER commit)
export RESTIC_REPOSITORY="azure:backups:/emr01"
export AZURE_STORAGE_ACCOUNT_NAME="<storage-account-name-from-B6-§5>"
export AZURE_STORAGE_ACCOUNT_KEY="<SET-IN-VAULT>"
export RESTIC_PASSWORD="<SET-IN-VAULT>"
```
GPG key prep (one-off, interactive — the key is passphrase-protected):
```bash
gpg --full-gen-key            # RSA 3072, Name-Real "NileCare Ops", email ops@nilecare.local
                              # passphrase = vault item; write it to /root/.gpg-pass (chmod 600)
gpg --export -a ops@nilecare.local > /root/ops-pub.asc
# PRIVATE key handling: export ONCE and store OFFLINE on the host PC + D:\NileCare\archive.
# It must never live in the git repo, on siem01, on cloud-dr, or in any screenshot.
# Keep the passphrase copy only in the team password vault.
```
Restore (drill = scenario 8): the loopback passphrase file must exist on the restore host — `/root/.gpg-pass` (or import the offline private key):
```bash
restic restore latest --target /tmp/emr-restore
gpg --batch --yes --pinentry-mode loopback --passphrase-file /root/.gpg-pass \
    --decrypt /tmp/emr-restore/var/backups/emr/openemr-<stamp>.sql.gz.gpg \
  | gunzip | mariadb openemr
mariadb openemr -e "SELECT COUNT(*) FROM patient_data;"   # record count → integrity evidence
```
systemd units:
```ini
# /etc/systemd/system/emr-db-backup.service
[Unit]
Description=NileCare EMR encrypted DB dump + immutable upload
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/emr-db-backup.sh
# /etc/systemd/system/emr-db-backup.timer
[Unit]
Description=Nightly EMR backup 01:00
[Timer]
OnCalendar=*-*-* 01:00:00
Persistent=true
[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload && systemctl enable --now emr-db-backup.timer
systemctl start emr-db-backup.service && journalctl -u emr-db-backup.service -n 20   # first-run evidence
```
## 2. Domain controllers — System State (Table 8.1 row 2)
```powershell
# dc01: nightly System State to \\dc02\Backups (B4 §6.4 created the share)
Install-WindowsFeature Windows-Server-Backup
schtasks /Create /TN "NileCare-SystemState" /SC DAILY /ST 23:00 /RU SYSTEM /RL HIGHEST /TR ^
 "wbadmin start systemstatebackup -backupTarget:\\10.10.70.12\Backups -quiet"
wbadmin start systemstatebackup -backupTarget:\\10.10.70.12\Backups -quiet   # first run now → evidence
# Verify: wbadmin get versions -backupTarget:\\10.10.70.12\Backups
```
- dc02 restic (Windows build, scoop/choco install restic) scheduled task 02:00 backing up `D:\Shares` + `D:\Backups` (holds dc01's System States) → `azure:backups:/dc02`:
```powershell
schtasks /Create /TN "dc02-restic" /SC DAILY /ST 02:00 /RU SYSTEM /TR ^
 "C:\ProgramData\chocolatey\bin\restic.exe backup D:\Shares D:\Backups --tag dc02"
```
Recovery of AD (drill): rebuild-or-restore decision tree in report §8.3; non-authoritative restore of an OU/object via Recycle Bin (already enabled, B4 §1.5) — demonstrate one object restore as §8 evidence.

## 3. PACS, portal, pfSense, Wazuh config (Table 8.1 rows 4–6)
- pacs01 (restic, timer 03:00): `restic backup /var/lib/orthanc /etc/orthanc /etc/nginx` → `azure:backups:/pacs01`; images are also re-downloadable (public datasets — documented, reduces retention pressure).
- portal01: config is Ansible-managed (repo = backup); add nightly `restic backup /etc/nginx` → `azure:backups:/portal01`.
- pfSense: **monthly manual** export `Diagnostics → Backup & Restore → Download XML` (encrypted with the config password) → copy to `E:\NileCare\repo\pfsense-config\` → restic push from siem01 → `azure:backups:/fw01`. No auto-cron (pfSense CE lacks it without a paid service) — calendar reminder documented.
- Wazuh + Ansible: `/opt/nilecare` + `/var/ossec/etc` committed to the **GitHub private repo** (template 5.7) + weekly `restic backup /opt/nilecare /var/ossec/etc` → `azure:backups:/siem01`. Wazuh indexes are NOT backed up (rebuildable from agents/logs — documented).

## 4. DR cutover path (report §8.3, measured in scenarios 7/8)
Pre-staged (do now, note in runbook):
1. pfSense Resolver: prepare (but leave disabled) host override `emr01.nilecare.local → 172.16.10.10`, TTL 60 s.
2. Temporary firewall rule (disabled): `CLINICAL/ADMIN → CLOUD_DR 443` (B0 §5 row 6) — enable during cutover only.
3. cloud-dr nginx: install the same OpenEMR TLS path (nginx 443 → php-fpm; DB = local replica) — configure while cloud-dr is ON so promotion is application-start only.
Cutover procedure (timed — exact timestamps go to Deliverable C):
```bash
# T0 "declare": date +%FT%T.%N | tee -a /root/dr-timeline.log   (run the equivalent timestamped log on the host, per C Scenario 7)
# 1. Verify replica current:  mariadb -e "SHOW REPLICA STATUS\G"   (Seconds_Behind_Source)
# 2. Stop replication and promote:
sudo mariadb -e "STOP REPLICA; RESET REPLICA ALL;"
# 3. Start application stack on cloud-dr (systemctl enable --now nginx php8.3-fpm)
# 4. Enable pfSense host override + CLINICAL/ADMIN→CLOUD_DR 443 rules; flush resolver cache
#    Diagnostics → DNS Resolver → Restart
# 5. T1 "validated": clinician login on https://emr01.nilecare.local (now cloud) + record lookup
```
RTO = T1 − T0 (placeholder until executed). Failback: reverse `CHANGE REPLICATION SOURCE` to emr01, re-enable on-prem overrides, reconcile the delta window (binlog replay documented in report text).

## 5. Ransomware recovery procedure (§8.4 — full simulation script in Deliverable C)
1. **Isolate**: power off the affected VM (lab) or block its IP at pfSense (production analogue); disable the compromised user account.
2. **Scope**: Wazuh FIM alerts + FSRM block events (`Get-WinEvent` on dc02) + last-good restic snapshot list.
3. **Restore**: from immutable Blob — the WORM policy prevents attacker deletion; `restic snapshots` → `restic restore <id>` to the share; verify GPG hashes of DB dumps before replay.
4. **Verify**: checksum comparison (restic restore output), test clinician login + file access, then re-enable VLAN with tightened rules; capture evidence.
5. **Less learned**: report §9.4 template text.

## 6. Business continuity — downtime procedures (§8.5, short text + table)
| Procedure | Trigger | Action |
|---|---|---|
| Paper workflow kit | EMR down > 30 min | Pre-printed intake forms per department (Annex), radiology film workflow |
| Read-only records | EMR app down, DB alive | Report queries from the `synthea_raw`/dumped copy on siem01 (read-only creds) |
| Comms tree | Any P1 incident | Incident commander → dept leads list (Ch10 roles) |

## 7. OVA cold archive (A §10 row 7)
```bash
# monthly, host PC (VMs powered off): export fw01, dc01, emr01 only
# VMware: File → Export to OVF → D:\NileCare\archive\<yyyy-mm>\<name>.ova  (single generation, ~28 GB)
```

## Evidence register (chapter 8)

| File | Content | Report section |
|---|---|---|
| replication-wire-test.png | row replicated ≤ seconds | §8.1 |
| emr-db-backup.log + restic-snapshots.png | nightly job + immutable repo | §8.1 |
| systemstate-wbadmin.png | `wbadmin get versions` | §8.1 |
| blob-immutability-policy.png | WORM policy (from B6) | §8.1 |
| dr-timeline.log | T0/T1 timestamps (scenario 7) | §8.2 |
| restore-verify.png | row counts after scenario-8 restore | §8.2 |

**Reminder:** measured RTO/RPO stay placeholders until Deliverable C scenarios run — never pre-fill them.
