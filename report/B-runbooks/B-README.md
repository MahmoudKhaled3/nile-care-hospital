# B-README — Runbook set overview (Deliverable B)

## Files and execution order

| Order | File | Report chapter | Runs on |
|---|---|---|---|
| 0 | `B0-prep-pfsense.md` | Ch3 (+ B prerequisites) | host, fw01 |
| 1 | `B4-windows.md` | Ch4 | dc01, dc02, win-client |
| 2 | `B5-linux.md` | Ch5 | emr01, pacs01, portal01, siem01, lin-client |
| 3 | `B6-cloud.md` | Ch6 | Azure (cloud-dr, Blob), fw01 VPN |
| 4 | `B7-security.md` | Ch7 | siem01, fw01, all agents |
| 5 | `B8-backup-dr.md` | Ch8 | emr01, dc02, pacs01, portal01, siem01, cloud-dr |

Team mapping (template Appendix D; full working briefs are in [`team/00-TEAM-PLAN.md`](../../team/00-TEAM-PLAN.md)): Mahmoud (lead integration, VMware/pfSense, Windows, final host) → B0, B4, pfSense side of B6, Windows side of B8; Youssef Al-Sayed (emr01, pacs01, portal01, lin-client, Ansible) → B5, B7 §5 and Linux/Ansible portions of B8 §§1/3; Iprahim Said (Azure DR) → B6 cloud side, Blob immutability, Azure Monitor; Mahmed Salh (synthetic data, Chapters 1–2, references, compliance review) → E Chapter 1, Chapter 2 using A and the template, and D review; Mahmed Shaker (siem01, Wazuh, evidence and report assembly) → B5 base siem01 and agent coordination, B7 §4, Wazuh config in B8 §3, C and Appendices C/D. Mahmoud owns Suricata on fw01. Everyone contributes their chapter's evidence.

Shaker hands over the base siem01 build before Gate 3 so Youssef can run Ansible on it. Shaker stages Wazuh alone and leads its integrated rollout in Gate 5. Mahmoud accepts the final result on his laptop after verification with the relevant host owners.

## Standing rules (from Deliverable A)
- Power-state cap **10.5 GB concurrent** (fw01 = 1 GB, see A §5 and the scenario phases in C); one clean-baseline snapshot per VM; scenario snapshots deleted after evidence lands in `E:\NileCare\evidence\`.
- Evidence files: `E:\NileCare\evidence\<fig-x.y-name>.png` (+ .txt/.csv logs) — matches the template Appendix C register.
- Scripts and playbooks live in `E:\NileCare\repo\` (git init day 1; GitHub private before B5 §7).
- Never pre-fill measured numbers (RTO/RPO, Nessus counts, Lynis index) — placeholders until the tests run.
- Versions are recorded at install time (pfSense CE — **version filled in after the Netgate Installer runs, expected 2.9.x**; media = Netgate Installer v1.2, official SHA256 verified · Server 2025 build · OpenEMR 7.0.x · Wazuh 4.x · **Ubuntu 24.04.3** — ISO verified against releases.ubuntu.com · win-client = **Windows 11 Education (Index 4), unactivated**, consumer 25H2 ISO, hash recorded) — "record exact version" notes are part of the evidence.
- VMware networking uses **VMnet10/12/13/14/15/16/17** (VMnet0–19 range only; VMnet11 = spare) — never VMnet20+.

## Gated execution order — STOP-AND-FIX at every gate
Do not start the next gate until the current one passes. Record each gate result in `E:\NileCare\evidence\gates.log`.

| Gate | Scope | Command(s) | Expected | If it fails |
|---|---|---|---|---|
| **1** | B0: VMnets + fw01 up, rules loaded, default-deny logging | Save the Virtual Network Editor image. Check HTTPS from the MGMT host. Use a static Guest client for local DNS and HTTPS egress, then verify a logged deny for Guest → Servers. | Editor matches B0 §1, GUI answers, Guest egress works, isolation deny appears in the firewall log. | Check NIC mapping, interface names, Guest static settings and rule order before building dc01. |
| **2** | B4: dc01 healthy (dcdiag, DNS, DHCP via relay) | `dcdiag /s:dc01`; `Resolve-DnsName dc01.nilecare.local`; on win-client: `ipconfig /all` | dcdiag all *passed*; DNS answers, Clinical lease `10.10.10.110` with AD DNS, and Guest lease with gateway/DNS `10.10.50.1`. Guest isolation remains intact | Fix DHCP relay destination, DNS forwarders, pfSense AD_PORTS rules; **do not run B5** |
| **3** | B5: Ansible dry-run, then real run | `ansible all -m ping`; `ansible-playbook --syntax-check *.yml` (or `ansible-lint`); then `ansible-playbook hardening.yml` | every Linux host *pong*; syntax check clean; playbook completes with no `failed=1` | Fix SSH keys/WinRM prep first; re-run until green; **do not touch Azure** |
| **4** | B6: tunnel up + replication running | `sudo ipsec statusall` on cloud-dr; `sudo mariadb -e "SHOW REPLICA STATUS\G"`; B8 §1.1 wire check | tunnel `INSTALLED`; `Replica_SQL_Running: Yes`; wire-test row arrives ≤ seconds | Phase-1/2 mismatch (AES256/SHA256/DH14), PSK, NSG UDP 500/4500, then resync per B6 §4 recovery; **no SIEM build yet** |
| **5** | B7: agents reporting + rules firing | `/var/ossec/bin/agent_control -l` (or dashboard); access PAT-VIP-001 → alert 100100 | all agents *Active*; VIP alert appears in dashboard | Check pfSense 1514/1515 allows, agent `<address>`, restart wazuh-agent; **no backup timers yet** |
| **6** | B8: backup + restore test | `systemctl list-timers \| grep emr-db`; `restic snapshots`; restore drill (Deliverable C scenario 8) | timer scheduled; snapshot in `azure:backups:/emr01`; restored row counts match | Verify `/etc/nilecare/backup.env` perms, `/root/.gpg-pass`, WORM container policy; re-run before declaring Ch8 done |

## Secrets & repo hygiene

**Rule: no secret value is ever committed, screenshotted, or pasted into a runbook.** All lab data is synthetic, but credential discipline is graded.

1. `.gitignore` (create in `E:\NileCare\repo\` on day 1):
   ```gitignore
   *.gpg
   *.pem
   *.key
   *.asc
   *.ovpn
   .env
   *vault.yml
   .vault-pass.txt
   backup.env
   tfstate/
   *.tfstate*
   evidence/screenshots-with-credentials/
   ```
2. Ansible secrets: `ansible-vault create group_vars/windows/vault.yml` → store `ansible_password:` (and any `winrm` TLS certs later). Run playbooks with `--ask-vault-pass` or a `vault_password_file` **outside the repo** (`~/.vault-pass.txt`, chmod 600). Never commit `.vault-pass.txt`.
3. Where secrets live:
   - Team password vault (KeePassXC database on the host PC, `E:\NileCare\repo\` excluded from sync) — the single source of truth.
   - `/etc/nilecare/backup.env` + `/root/.gpg-pass` on emr01 (chmod 600, root-only) — deployment copies, not sources.
   - GPG private key (`ops@nilecare.local`): **offline only** — one export on the host PC and one copy in `D:\NileCare\archive\`; never in the repo, never on siem01/cloud-dr, never inside a VM snapshot.
4. Sharing rules:
   - **May share:** runbooks, playbooks without vault files, evidence screenshots *after* checking no password/PSK/key is visible, scan exports.
   - **May not share:** vault database, vault passwords, `.gpg-pass`, `backup.env`, `ipsec.secrets`, exported private keys, Wazuh dashboard credentials, `ops-priv.asc`.
   - Screenshots: blur or crop any credential field before it enters `E:\NileCare\evidence\`.

## Figure placeholder → evidence mapping (report template)

| Template figure | Evidence file | Produced in |
|---|---|---|
| Figure 3.1 Overall network architecture | fig-3.1-network-diagram.png | B0 §1–§8 (draw.io) |
| Figure 4.1 ADUC OU structure | fig-4.1-aduc-ou-structure.png | B4 §4 |
| Figure 4.2 DHCP scopes per VLAN | fig-4.2-dhcp-scopes.png | B4 §3 |
| Figure 4.3 gpresult /r on client | fig-4.3-gpresult.png | B4 §9 |
| Figure 4.4 FSRM file screen blocking | fig-4.4-fsrm-file-screen.png | B4 §6.5 |
| Figure 5.1 OpenEMR login over HTTPS | fig-5.1-openemr-login-https.png | B5 §2.2 |
| Figure 5.2 Domain user login on Linux | fig-5.2-domain-user-login-linux.png | B5 §5 |
| Figure 6.1 Cloud network diagram | fig-6.1-cloud-network-diagram.png | B6 §1 |
| Figure 6.2 VPN tunnel status both ends | fig-6.2-vpn-tunnel-status.png | B6 §3.3 |
| Figure 7.1 Firewall rules per interface | fig-7.1-firewall-rules-per-interface.png | B0 §5 |
| Figure 7.2 Wazuh dashboard | fig-7.2-wazuh-dashboard.png | B7 §4 |
| Figures 9.1–9.9 scenario evidence | per Deliverable C | C scenarios |

## Known honest limitations carried into the report (already in A §1/§13)
- dc02 dual-role (DC+file server) — accepted, 10.5 GB RAM cap.
- win-client = **Windows 11 Education (Index 4), unactivated** — the team's consumer 25H2 ISO ships no Enterprise; Education enforces AppLocker and unactivated licensing never expires (watermark only). The Win10 22H2 **Pro** VM (EOS Oct 2025) is preserved untouched as a backup host and would support AppLocker audit-mode only. **Mandatory early enforcement test** (B0 §2 step 9) gates all AppLocker evidence; fallback = Windows 11 Enterprise Evaluation (A §15.2).
- Win11 client requires EFI + Secure Boot + virtual TPM 2.0 (Workstation "fast" VM encryption) or the LabConfig bypass, and 4 GB RAM — follow the scenario power phases in C.
- pfSense CE is installed via the **Netgate Installer v1.2** (media SHA256 verified against Netgate's official checksums file); the installed pfSense version is recorded after install (expected 2.9.x); the installer's screens are not documented step-by-step in B0 §3 (described from official docs, not seen first-hand).
- WSUS replaced by staged deferral GPOs; WSUS documented as production alternative.
- Nessus Essentials on the host PC replaces OpenVAS (host cannot afford a GVM VM); tool change stated in §9.3.
- Ansible co-located on siem01; WinRM over HTTP inside the segmented lab; production = TLS 5986.
- pfSense WAN behind home-router NAT: OpenVPN inbound test needs a home-router port-forward, else "tested from LAN only" is stated.
- VPN Gateway avoided (cost) — strongSwan on cloud-dr; dynamic home IP caveat documented.
- Microsoft publishes no SHA256 for the WS2025 Eval ISO; provenance = official Evaluation Center filename + two identical local copies (hashes recorded in A §15.1).
