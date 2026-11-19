# Deliverable A — Revised Design Package
**Secure Hybrid Enterprise Infrastructure — Nile Care Hospital**
New Cairo Technological University | Server Administration Final Project | Domain: `nilecare.local`
Version 1.0 — 2026-11-18. All patient data synthetic (Synthea) + public DICOM datasets.

---

## 0. Reference Host Baseline (measured 2026-11-18)

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation Pro 17 (free for personal use) |
| Host OS | Windows 10/11 x64 (build 26300) |
| RAM | 15.9 GB |
| CPU | AMD Ryzen 5 4500 — 6 cores / 12 threads |
| E: | 238.5 GB SSD, **87.2 GB free** → VM storage (`E:\NileCare\vms`) |
| D: | 930 GB HDD, **157.1 GB free** → ISO media (`D:\NileCare\iso`) + cold archive (`D:\NileCare\archive`) |
| C: | 14.9 GB free — leave untouched |
| Existing VM | `E:\Tools\WIN10` — Windows 10 22H2 **Pro** (edition confirmed 2026-11-18 via `wmic os get Caption`) — **preserved untouched as backup host, NOT win-client** (Pro cannot enforce AppLocker). 23 GB folder incl. "Snapshot 1" (4 GB memory) — do not modify |

> The table records the desktop used for sizing. Mahmoud's laptop is the final integration and demonstration host, with a reported ninth-generation Intel Core i7 and 16 GB RAM. Usable RAM, available disk space, virtualization status, and the VMware version on that laptop remain pending verification. Apply the 10.5 GB concurrent VM RAM cap and confirm the planned groups during rehearsal.

**Feasibility verdict (honest):**
- Provisioned RAM across all 9 on-host VMs = **17.75 GB (18,176 MiB)** — never run everything at once. **Cap concurrent VM RAM at 10.5 GB** so the 15.9 GB host always keeps ≥5.4 GB. Set VMware *Edit → Preferences → Memory → "Allow some virtual machine memory to be swapped"* so allocation never hard-fails near the cap. siem01 = 4 GB and win-client (Windows 11 Education, unactivated) = 4 GB, so use the phase sets in §5 and Deliverable C.
- **Reference host datastore split.** E: SSD holds fw01, dc01, emr01, siem01 and portal01. D: HDD holds dc02, pacs01, the fresh Windows 11 win-client VM and lin-client, plus ISOs and the archive. Record actual datastore paths on the final laptop.
- **E: headroom:** measured 82.8 GB free. The backup VM (`E:\Tools\WIN10`, ~23 GB used) is **untouched — including "Snapshot 1"**. Consumption: hot-set growth ≈53.5 GB (43.5 used + 10 snapshot delta) → projected **≈29.3 GB free ✓** (win-client moved to D:, see §5). Optional knob: deleting "Snapshot 1" on the backup VM would add ≈4.9 GB (≈34.2 GB) — **not recommended now, keep the backup pristine**.
- Snapshot policy: **one clean-baseline snapshot per VM** (post-build, post-hardening); scenario snapshots are deleted immediately after evidence is captured to `E:\NileCare\evidence\`; disks compacted monthly.
- **win-client = Windows 11 Education (unactivated) — decision (a) updated 2026-11-18:** the existing VM's edition was confirmed **Windows 10 Pro** (no AppLocker enforcement). The team's downloaded media is the consumer **25H2 ISO** (`Win11_25H2_English_x64.iso`), which ships **Home/Education/Pro/Pro Education/Pro for Workstations** (no Enterprise) → **win-client = Windows 11 Education (setup Index 4, verify per DISM)**, running **unactivated**. Education enforces AppLocker; unactivated = watermark + personalisation limits only, **no expiry** (better for a semester than the 90-day Enterprise Eval). Fallback if the mandatory early enforcement test fails: Windows 11 Enterprise Evaluation (steps in §15.2). `E:\Tools\WIN10` (Pro) preserved untouched as backup host. RAM impact (2.5 → 4 GB) absorbed by the sets in §5.

## 1. Design Decision Log (the 8 fixes → decisions taken)

| # | Flaw | Decision |
|---|---|---|
| 1 | No server VLAN | **VLAN 70 "Servers" 10.10.70.0/24** added (dc01, dc02, emr01). Explicit allow rules to DCs (AD ports, restricted RPC range 49152–49252), to emr01 (443 only). DHCP relay on pfSense for VLAN 10/20/50 → dc01 during Gate 2. Medical VLAN 30 → DCs: **not required — denied with documented rationale** (§7). |
| 2 | Wazuh agents blocked toward MGMT | Explicit **Limited** allows: agents (VLAN 10/20/30/40/70) → siem01 **TCP 1514/1515** only; Linux servers → siem01 **TCP 514** (rsyslog). "Log only" redefined as **"Deny (logged)"** — explicit logged block rules; default deny preserved. Table 3.4 rebuilt + companion Table 3.5 with exact ports. |
| 3 | emr01 role conflict | Split: **emr01** = OpenEMR + MariaDB (VLAN 70, Ubuntu 24.04); **pacs01** = Orthanc (VLAN 30, Ubuntu 24.04). DICOM 104/4242 stays inside VLAN 30; DICOMweb/HTTPS 443 is available to Clinical/Admin, plus the documented MGMT administration path. Inventory (3.3), asset inventory (2.2), backup matrix (8.1) updated. |
| 4 | Synthea ≠ DICOM | PACS seeded from **public anonymised DICOM datasets** (Orthanc sample dataset / TCIA). Declaration rewritten (§14). Synthea used for EMR clinical records only. |
| 5 | Linux stack inconsistency | **Ubuntu Server 24.04 LTS everywhere** (all servers + Linux test client). UFW + AppArmor + Fail2ban. Table 5.7 uses the same Ubuntu tools. |
| 6 | EMR RPO (15 min) not covered | **MariaDB async replication** emr01 → cloud-dr over the site-to-site VPN (the replica pulls binlogs, target RPO 15 min) **+** nightly `mysqldump | gzip | gpg` → Azure Blob with immutability policy. RTO/RPO measurement method (exact timestamps/commands) specified for Scenarios 7–8 in Deliverable C. |
| 7 | dc02 dual-role (DC + file server) | **Kept dual-role** — with 16 GB host RAM an fs01 VM (+2 GB) would break the concurrency budget. Justified in Limitations (11.3); mitigations: FSRM file screening, hardened GPO, no interactive logon rights on dc02. If the team later gets ≥32 GB, add fs01. |
| 8 | RAM budget / Zabbix | VM sizing in §5 (**17.75 GB provisioned** — siem01 4 GB for the Wazuh stack, win-client 4 GB as a Windows 11 Education client; concurrency cap **10.5 GB**, phase-based power policy). **Zabbix dropped** — Wazuh only. Cloud free tier (Azure for Students) hosts **cloud-dr + Blob storage only**. |
| 9 | win-client build cost / licensing | Edition check executed on the existing VM (`E:\Tools\WIN10`): **Windows 10 Pro** — AppLocker *enforcement* impossible. Media reality: the team downloaded the consumer **Win11 25H2 ISO** (Education/Pro/Home present, **no Enterprise**) → **decision (a) final form: win-client = Windows 11 Education (setup Index 4), unactivated, on D:** — 2 vCPU / 4 GB / 64 GB thin / EFI + Secure Boot + TPM 2.0 (or LabConfig bypass) / NIC1 → VMnet10 / hostname `WIN-CLIENT01`. **Mandatory early test: a local AppLocker deny rule must actually block a test exe while unactivated — if it does not, fall back to Windows 11 Enterprise Evaluation (§15.2).** The Win10 Pro VM is **preserved untouched as a backup host**; EOS (Oct 2025) noted in Limitations 11.3. Follow the phased power schedule in §5. |

**Other confirmations:** firewall = **pfSense CE** (matches template Technology table); Medical→MGMT vs MGMT→Medical asymmetry is intentional (MGMT manages; the reverse is limited to agent telemetry ports); hypervisor = VMware Workstation Pro 17.

## 2. VLAN and IP Addressing Plan — Table 3.1

| VLAN | Name | Subnet | Gateway | DHCP | Purpose |
|---|---|---|---|---|---|
| 10 | Clinical | 10.10.10.0/24 | 10.10.10.1 | dc01 (relay) `.100–.200` | Doctors, nurses, EMR workstations |
| 20 | Admin | 10.10.20.0/24 | 10.10.20.1 | dc01 (relay) `.100–.200` | Billing, HR, reception, domain Linux tools |
| 30 | Medical/PACS | 10.10.30.0/24 | 10.10.30.1 | Static | Imaging modalities (simulated), pacs01 — **not domain-joined** |
| 40 | DMZ | 10.10.40.0/24 | 10.10.40.1 | Static | Patient portal (portal01) |
| 50 | Guest | 10.10.50.0/24 | 10.10.50.1 | Gate 1 static. Later: dc01 via relay, `.50–.250` | Internet only |
| 60 | MGMT | 10.10.60.0/24 | 10.10.60.1 | Static | siem01, administrator host adapter |
| 70 | **Servers (NEW)** | 10.10.70.0/24 | 10.10.70.1 | Static | dc01, dc02, emr01 |
| 90 | OpenVPN tunnel | 10.10.90.0/24 | pfSense | Virtual | Remote clinicians (MFA, UDP 1194 on WAN), no VM |
| Cloud | vnet-dr | 172.16.0.0/16 (snet-dr 172.16.10.0/24) | — | Azure | cloud-dr DR replica |

## 3. VMware Network Map — Table 3.2 (Workstation 17, Virtual Network Editor)

VMware host-only networks cannot carry 802.1Q tags, so each VLAN is a dedicated vmnet; pfSense gets one NIC per VLAN. The inter-VLAN routing/firewall model is identical to a tagged switch deployment — document this substitution in §3.2 of the report. VMware supports **VMnet0–VMnet19 only**, hence the remap in v1.3 (VMnet11 intentionally unused as a spare); IPs, subnets, rules and ports are unchanged.

| VMnet | Type | Carries | Subnet | Host-side DHCP |
|---|---|---|---|---|
| VMnet8 | NAT (default) | WAN (fw01 + all VM egress) | 192.168.x.0/24 | on (NAT) |
| VMnet10 | Host-only | VLAN 10 Clinical | 10.10.10.0/24 | **off** (dc01) |
| VMnet12 | Host-only | VLAN 20 Admin | 10.10.20.0/24 | **off** (dc01) |
| VMnet13 | Host-only | VLAN 30 Medical | 10.10.30.0/24 | off |
| VMnet14 | Host-only | VLAN 40 DMZ | 10.10.40.0/24 | off |
| VMnet15 | Host-only | VLAN 50 Guest | 10.10.50.0/24 | off (VMware DHCP stays off) |
| VMnet16 | Host-only | VLAN 60 MGMT | 10.10.60.0/24 | off |
| VMnet17 | Host-only | VLAN 70 Servers | 10.10.70.0/24 | off |

- fw01 NICs: `WAN=VMnet8`, `CLINICAL=VMnet10`, `ADMIN=VMnet12`, `MEDICAL=VMnet13`, `DMZ=VMnet14`, `GUEST=VMnet15`, `MGMT=VMnet16`, `SERVERS=VMnet17` (8 vNICs).
- Disable VMware DHCP on every dedicated lab host-only vmnet and enable the host virtual adapter only on **VMnet16** among those networks. Verify the settings on the final laptop during Gate 1.
- The design gives the host one lab adapter on VMnet16 (MGMT) with static `10.10.60.100/24`, no default gateway, and a static route `10.10.0.0/16 via 10.10.60.1` through pfSense. The route was verified on the reference desktop. Verify it on the final laptop during Gate 1. The host captures evidence and scans systems. Scenario 6B uses its browser for the MGMT-origin test.
- pfSense webGUI: `https://10.10.60.1` (MGMT interface, from host or siem01).

## 4. Static IP Allocation — Table 3.3

| Host | OS | Role | IP | Notes |
|---|---|---|---|---|
| fw01 | pfSense CE — **version recorded after install** (expected 2.9.x; media = Netgate Installer v1.2, official SHA256 verified) | Firewall, VPN, Suricata, DHCP relay, NTP | `.1` on all VLANs | WAN via VMnet8 NAT |
| dc01 | Windows Server 2025 Eval (DE) | Primary DC, DNS, DHCP, PDC/NTP | 10.10.70.11 | DNS → 127.0.0.1 + dc02; forwarders 1.1.1.1/9.9.9.9 |
| dc02 | Windows Server 2025 Eval (DE) | Secondary DC, DNS, **file server + FSRM** | 10.10.70.12 | Dual-role — justified in 11.3 |
| emr01 | Ubuntu Server 24.04 LTS | OpenEMR 7.x (nginx+PHP-FPM) + MariaDB | 10.10.70.21 | AD-joined via SSSD; replicates to cloud-dr |
| pacs01 | Ubuntu Server 24.04 LTS | Orthanc + DICOMweb (nginx TLS) | 10.10.30.21 | Standalone auth; AET allowlist |
| portal01 | Ubuntu Server 24.04 LTS | nginx reverse proxy (patient portal) | 10.10.40.10 | Only internet-exposed host |
| siem01 | Ubuntu Server 24.04 LTS | Wazuh all-in-one **+ Ansible control node** | 10.10.60.21 | Ansible co-located (lab consolidation — note in 11.3) |
| win-client | **Windows 11 Education** (unactivated — Education enforces AppLocker) | Clinical test workstation, AppLocker (enforced) | DHCP reservation **10.10.10.110** (ClientId = **`<VERIFY-MAC-AFTER-NIC-SWAP>`** — record the real MAC from VM Settings → Network Adapter → Advanced after the NIC is configured) | VLAN 10 (VMnet10); Wazuh agent; built on **D:** from the consumer 25H2 ISO (setup Index 4); hostname `WIN-CLIENT01`; the Win10 Pro VM at `E:\Tools\WIN10` remains as untouched backup |
| lin-client | Ubuntu Server 24.04 LTS (min.) | Multi-role test client (approved connectivity checks / dcmtk) | per homing: VLAN 50 = **10.10.50.60**, VLAN 60 = **10.10.60.150**, VLAN 30 = **10.10.30.60** | Single NIC re-homed per test — documented lab convenience (C §0.5) |
| cloud-dr | Ubuntu Server 24.04 LTS (Azure B1s) | DR replica: MariaDB + strongSwan + restic | 172.16.10.10 | Public IP; Wazuh agent → siem01 via VPN |
| (host) | Windows | Administrator workstation / evidence capture | 10.10.60.100 (VMnet16, route 10.10.0.0/16 via 10.10.60.1) | Evidence workstation; approved MGMT browser exception in Scenario 6B |

**lin-client re-homing schedule (honest lab simplification):** Scenario 4 (Guest segmentation validation) → VLAN 50 · Scenario 5 (SSH protection event validation) → VLAN 60 · DICOM modality simulation → VLAN 30 (dcmtk modality simulation → pacs01). A real hospital uses separate devices; note in 11.3.

## 5. VM Inventory and Sizing (companion to Table 3.3)

| VM | vCPU | RAM | Disk (thin) | Est. used | Datastore | Power policy |
|---|---|---|---|---|---|---|
| fw01 | 1 | 1 GB | 16 GB | ~3 GB | **E:** | Always on |
| dc01 | 2 | 2.5 GB | 24 GB | ~18 GB | **E:** | Always on |
| dc02 | 2 | 2 GB | 24 GB | ~18 GB | **D:** | **Off by default** — on for DC-replication / FSRM / backup labs |
| emr01 | 2 | 2 GB | 16 GB | ~7 GB | **E:** | On during Ch5–8 labs |
| pacs01 | 1 | 1 GB | 12 GB | ~4.5 GB | **D:** | On for imaging labs (moved off E: to protect the ≥15 GB rule) |
| portal01 | 1 | 768 MB | 8 GB | ~3.5 GB | **E:** | On for DMZ/scenario labs |
| siem01 | 2 | **4 GB** | 24 GB | ~12 GB | **E:** | Base OS and Ansible at Gate 3. Wazuh integration at Gate 5. Power on for the relevant test phases. |
| win-client | 2 | **4 GB** | 64 GB | ~25–30 GB (Win11 Education + updates) | **D:** (fresh Win11 Education VM) | **Off by default** — on for Ch4/9 labs |
| lin-client | 1 | 512 MB | 8 GB | ~4 GB | **D:** | On per scenario |
| **Totals** | **14 vCPU** | **17.75 GB (18,176 MiB) prov.** | **196 GB thin (E: 88 / D: 108)** | **~95 GB** | E:/D: | **≤ 10.5 GB concurrent** |

**Concurrency sets (cap = 10.5 GB; host 15.9 GB keeps ≥5.4 GB):**
- Windows labs: fw01 + dc01 + dc02 + win-client = **9.5 GB**.
- Linux application setup: fw01 + emr01 + pacs01 + portal01 = **4.75 GB**.
- SIEM and application logs: fw01 + dc01 + emr01 + siem01 = **9.5 GB**. Use the phase schedule in Deliverable C when a client is required.
- DR backup and restore: fw01 + dc02 + emr01 + siem01 = **9.0 GB**.
- DR client validation: fw01 + dc01 + emr01 + win-client = **9.5 GB**, dropping to **7.5 GB** after emr01 is powered off.
- All nine VMs require **17.75 GB**. Keep each phase at or below **10.5 GB**.

**Reference host disk placement and estimated budget (2026-11-18):**

| Drive | Role | At build start | Planned consumption | Projected free |
|---|---|---|---|---|
| E: SSD | Hot VM set (fw01, dc01, emr01, siem01, portal01) = 88 GB thin / ~43.5 GB used + ~10 GB snapshot delta; **backup VM untouched (`E:\Tools\WIN10`, 23 GB used — already counted)** | 82.8 GB (**with** "Snapshot 1" kept — recommended) | +53.5 GB (hot set only — win-client is on D: now) | **≈29.3 GB ✓** — deleting "Snapshot 1" on the backup would give ≈34.2 GB (not recommended; keep the backup pristine) |
| D: HDD | Cold VM set (dc02, pacs01, lin-client, **win-client**) = 108 GB thin / ~54.5 GB used (dc02 18 + pacs01 4.5 + lin 4 + win-client ~28) + ~3.5 GB win-client snapshot delta + ISOs 24.1 GB (WS2025 6.01, Win11 25H2 7.74, Win10 6.14, Ubuntu 3.30, netgate-installer .gz 0.34 + .iso 1.06) + monthly OVA exports ~28 GB | 157.1 GB | +110.0 GB | **≈47.1 GB ✓** |

**Space note:** with win-client moved to D: (decision (a)) the E: rule (≥15 GB) holds with margin even with the backup VM fully intact. Knobs if E: ever dips: delete "Snapshot 1" on the backup (+4.9 GB), siem01 indexer retention 30→14 d.

**Backup VM facts (`E:\Tools\WIN10`, measured + edition-confirmed 2026-11-18):** Windows 10 **Pro** (`wmic os get Caption`); 23.0 GB folder (base split-sparse disk 15 GB provisioned / ~13.9 GB used = 93% full + active delta ~0.8 GB + `Snapshot1` incl. 4 GB memory file); 2 vCPU / 4 GB configured, EFI, VMware Tools installed, NIC1 MAC `<LEGACY-MAC-NOT-USED>`. **Status: preserved untouched as backup/spare host** — not joined to the domain, not scanned, no lab traffic; it can serve as an emergency clinical workstation (AppLocker audit-mode only).

**dc02 vs fs01 (fix 7):** kept dual-role. Adding fs01 (2 vCPU / 2 GB / 24 GB) would push the maximum compliant concurrency set past the 10.5 GB cap. Limitations §11.3 text: *"dc02 combines the secondary DC and file-server roles to stay within the 16 GB host budget; in production these roles are separated because a file-server compromise would otherwise expose AD DS services hosted on the same host."*

## 6. Inter-VLAN Policy Matrix — Table 3.4 (rebuilt)

Legend: **Allow** = unrestricted | **Limited** = only ports in Table 3.5 | **Deny (logged)** = explicit logged block (replaces "Log only") | same-VLAN traffic unaffected. Internet web rules use TCP 80/443 and exclude the firewall itself, RFC1918 addresses and all other lab ranges. Guest receives the local DNS and DHCP services described below.

| From \ To | Clin 10 | Admin 20 | Med 30 | DMZ 40 | Guest 50 | MGMT 60 | Srv 70 | Internet |
|---|---|---|---|---|---|---|---|---|
| **Clinical 10** | — | Deny | Limited (443) | Limited (443) | Deny | **Limited (1514/1515)** | Limited (AD, EMR 443) | Limited (public TCP 80/443) |
| **Admin 20** | Deny | — | Limited (TCP 443 to pacs01) | Limited (443) | Deny | **Limited (1514/1515)** | Limited (AD, SMB, EMR 443) | Limited (public TCP 80/443) |
| **Medical 30** | Deny | Deny | — | Deny | Deny | **Limited (1514/1515, 514)** | **Deny** (rationale §7) | Limited (apt alias) |
| **DMZ 40** | Deny | Deny | Deny | — | Deny | **Limited (TCP 1514/1515 and portal01 syslog TCP 514)** | Limited (443→emr01) | Limited (apt alias) |
| **Guest 50** | Deny | Deny | Deny | Deny | — | Deny | Deny | Limited (public TCP 80/443) |
| **MGMT 60** | Limited (management ports) | Limited (management ports) | Limited (management ports) | Limited (management ports) | Deny | — | Limited (management ports) | Limited (public TCP 80/443 and DNS/NTP) |
| **Servers 70** | Deny | Deny | Deny | Deny | Deny | **Limited (1514/1515, 514)** | — | Limited (updates, backup 443) |
| **Internet →** | Deny | Deny | Deny | **Limited (443→portal01)** | Deny | Deny | Deny | — |

Notes: the asymmetry Medical↔MGMT is intentional (MGMT manages; Medical only returns agent/syslog telemetry). Remote clinicians (OpenVPN 10.10.90.0/24) get the same "Limited" set as Clinical toward VLAN 70. ICMP: echo allowed within a VLAN and from MGMT only.

## 7. Allowed-Flow Port Table — Table 3.5 (companion to 3.4)

| # | Flow | Ports | Purpose |
|---|---|---|---|
| 1 | Clin/Admin → dc01, dc02 | UDP+TCP 53 · TCP/UDP 88 · TCP/UDP 464 · TCP/UDP 389 · TCP 636 · TCP 3268 · TCP 445 · TCP 135 + TCP 49152–49252 · UDP 123 | DNS, Kerberos, kpasswd, LDAP/LDAPS, GC, SMB, RPC (restricted range, footnote), NTP |
| 2 | pfSense relay → dc01 | UDP 67 | DHCP for VLAN 10/20/50 (client 68 → relay → server 67). Guest has no direct access to dc01. |
| 3 | Clin/Admin/MGMT → emr01 | TCP 443 | OpenEMR over HTTPS |
| 4 | Clin/Admin → pacs01 | TCP 443 | DICOMweb / Orthanc web viewer (TLS via nginx) |
| 5 | Clin/Admin → portal01 | TCP 443 | Patient portal (staff testing) |
| 6 | Internet → portal01 | TCP 443 (WAN port-forward) | Public patient portal |
| 7 | DMZ (portal01) → emr01 | TCP 443 | Portal → EMR application path |
| 8 | All agent VLANs (10/20/30/40/70) → siem01 | TCP 1514, TCP 1515 | Wazuh events + enrolment — the "Limited" telemetry exception |
| 9 | pacs01, portal01, emr01 → siem01 | TCP 514 | rsyslog forwarding |
| 10 | MGMT segment (10.10.60.0/24 — host .100, siem01 .21, lin-client .150 for scenario 5) → Linux (emr01, pacs01, portal01, lin-client) | TCP 22 | SSH / Ansible (Fail2ban guards the targets) |
| 11 | MGMT → Windows (dc01, dc02, win-client) | TCP 3389, TCP 5985 | RDP + WinRM (Ansible) |
| 12 | MGMT → all web UIs | TCP 443 | OpenEMR/Orthanc/Wazuh dashboard/pfSense GUI |
| 13 | Medical (30) → Servers (70) | **none — Deny (logged)** | Modalities are not domain-joined (rationale below) |
| 14 | Servers (70) → Internet | TCP 80/443 (Windows Update, apt), TCP 443 (Azure Blob endpoints), UDP 123 | Patching + encrypted off-site backup |
| 15 | Medical/DMZ → Internet | TCP 80/443 to pfSense alias `UBUNTU_ARCHIVE` only | Patching via allowlisted Ubuntu mirrors |
| 16 | Guest (50) → any internal | **none — Deny (logged)** | Isolation (Scenario 4) |
| 17 | cloud-dr (172.16.10.10) → emr01 (10.10.70.21) via IPsec | TCP 3306 | Replica initiates the connection. Binlog data returns over that connection. Target RPO 15 min. |
| 18 | cloud-dr → siem01 (via IPsec) | TCP 1514 | Wazuh agent from DR site |
| 19 | Remote clinicians (10.10.90.0/24) → Servers | Row 1 set + emr01 443 | VPN clinicians = clinical rules |

**Footnotes:** (a) RPC dynamic range restricted on dc01/dc02 to **49152–49252** via `HKLM\SOFTWARE\Microsoft\Rpc\Internet` (`Ports=49152-49252`, `PortsInternetAvailable=N`, `UseInternetPorts=N`) — configured in Ch4 runbook; without it the range is 49152–65535. (b) DC↔DC replication and all same-VLAN traffic never crosses pfSense. (c) IPsec phase-2 selectors: `10.10.70.0/24 ↔ 172.16.10.0/24` (flow 17) and `10.10.60.0/24 ↔ 172.16.10.0/24` (flow 18).

**Medical VLAN 30 → DCs: why not (decision, fix #1).** Modalities (simulated by pacs01/lin-client and dcmtk) are appliance-like systems: they are not domain-joined, use local auth, and are managed by Ansible over the MGMT path. Joining them to AD would require opening Kerberos/LDAP/SMB from the highest-compromise-likelihood segment into the domain controllers, expanding the AD attack surface for no functional gain — OpenEMR performs clinician authentication against AD server-side (emr01), so clinical identity is enforced without VLAN 30 touching the DCs. Deny is logged.

**DHCP relay (fix #1):** Gate 1 uses a static Guest client at 10.10.50.60/24 (gateway and DNS 10.10.50.1). At Gate 2, leave the pfSense DHCP server disabled everywhere and relay CLINICAL, ADMIN and GUEST to dc01 (10.10.70.11). Guest scope: .50–.250, gateway/DNS .1, 2-hour lease, no AD DNS or domain option. Clinical/Admin retain their .100–.200 scopes and AD DNS. Guest can reach its local DHCP relay and resolver. Forwarded Guest connections to Servers remain blocked and logged.
**DNS:** domain clients → dc01/dc02 (AD-integrated zones, forwarders 1.1.1.1/9.9.9.9); non-domain segments (30/40/50) → pfSense Resolver (unbound) with host overrides for `emr01`/`pacs01`/`portal01` so non-domain hosts resolve internal names.
**NTP:** dc01 (PDC) syncs to `pool.ntp.org` (`w32tm`), serves the AD domain; pfSense NTP service serves VLANs 30/40/50 (`chrony` targets `10.10.30.1/40.1/50.1`).

## 8. Technology Selection and Justification — Table 3.6

| Function | Choice | Why chosen (real justification, not filler) |
|---|---|---|
| Virtualization | VMware Workstation Pro 17 | Free for personal use (since 2024); snapshots/clones are required by the 180-day eval + DR tests; host-only vmnets model VLANs 1:1; runs on the existing Windows host — no second machine needed. |
| Firewall / Router | pfSense CE — **version recorded after install** (expected 2.9.x) via the **Netgate Installer v1.2** | Netgate retired the classic CE ISO mirror; the official path is the Netgate Installer ($0, store) which fetches the latest packages over the internet at install time (docs.netgate.com/install/netinstaller.html). Media verified: `netgate-installer-v1.2-RELEASE-amd64.iso.gz` SHA256 `184514FE…57955` **matches the official checksums file** (netgate.com/hubfs/pfSense-plus-installer-checksums.txt). Open source, per-interface ruleset, built-in DHCP relay, IPsec for the S2S VPN, OpenVPN+TOTP for remote clinicians, Suricata package. |
| IDS/IPS | Suricata (ET open rules) on fw01 | Inline with the firewall appliance; monitors WAN + internal segments; alerts forwarded to Wazuh via syslog — no extra appliance on a 16 GB host. |
| SIEM / XDR | Wazuh 4.x all-in-one (siem01) | Single tool = agent management, FIM, vulnerability detection, active response, dashboards — replaces Zabbix (dropped) to fit RAM budget; agents need only 1514/1515, compatible with segmentation. |
| Directory / Identity | Windows Server 2025 Evaluation (build 26100) | 180-day eval matches the semester; AD DS+DNS+DHCP+GPO in one; **snapshot before every structural change** (pre-promo, pre-GPO). |
| Client | **Windows 11 Education** (consumer 25H2 ISO, Index 4, **unactivated**) | AppLocker *enforcement* requires Enterprise/Education; the team's downloaded consumer ISO ships Education but **no Enterprise**, so the client is Win11 **Education** — Education enforces AppLocker, and unactivated licensing costs nothing (watermark + personalisation limits only, **no expiry** — friendlier for a semester than the 90-day Enterprise Eval). Existing Win10 Pro VM preserved untouched as backup host (EOS Oct 2025 noted). Win11 needs EFI + Secure Boot + TPM 2.0 (Workstation 17 virtual TPM with fast VM encryption, or LabConfig bypass) and 4 GB RAM — reflected in §5. |
| Linux base | Ubuntu Server 24.04 LTS (24.04.3 media verified) | One distro everywhere (servers + test client): 5-year support, AppArmor + UFW + Fail2ban + `unattended-upgrades`; largest documentation base for Ansible/Wazuh/OpenEMR. ISO SHA256 verified against releases.ubuntu.com. |
| EMR | OpenEMR 7.x + MariaDB | Open source, ONC-certified hospital workflows; runs on nginx+PHP-FPM; LDAP auth against AD; MariaDB gives binlog-based replication for the 15-min RPO. |
| Imaging / PACS | Orthanc + DICOMweb plugin | Lightweight PACS with REST API; `orthanc-dicomweb` in Ubuntu repos; seeded from public DICOM samples (TCIA/Orthanc), never Synthea. |
| Patient portal | nginx reverse proxy (portal01) | Only internet-exposed host; TLS termination, single 443 path to emr01; matches template §5.4 DMZ design. |
| Automation | Ansible (control node on siem01) | Agentless over SSH/WinRM; idempotent playbooks (hardening/users/packages/wazuh-agent) — co-location with siem01 documented as a lab consolidation. |
| Cloud DR | Azure for Students ($100 credit) | B1s (1 vCPU/1 GB) VM: strongSwan policy IPsec to pfSense (avoids the paid Azure VPN Gateway), MariaDB replica, restic to Blob; Blob immutability policy = version-worm storage on student terms. |
| Backup / crypto | restic + GPG + systemd timers | restic: client-side AES-256, dedup, native Blob backend; GPG for raw DB dumps; timers replace cron for journald evidence. |
| Linux ↔ AD identity | SSSD + `realm join` | Domain login + sudo rule for GRP-IT-Admins on emr01 (template §5.5); OpenEMR does LDAP auth server-side. |
| Security validation | Nessus Essentials, Lynis, dcmtk | Nessus for before/after vulnerability management, approved connectivity checks for segmentation validation, dcmtk simulates modalities, Synthea generates clinical records only. |
| MFA | pfSense User Manager TOTP (OpenVPN) | Native TOTP = real MFA for remote clinicians at zero cost — honest lab-scale answer; production would use Entra ID/RADIUS. |

## 9. Updated Asset Inventory — Table 2.2

| Asset | Type | Owner | Criticality |
|---|---|---|---|
| emr01 (OpenEMR + MariaDB) | Application/DB | Clinical IT | **Critical** |
| dc01, dc02 (AD DS, DNS, DHCP) | Infrastructure | IT | **Critical** |
| fw01 (pfSense) | Infrastructure | IT | **Critical** |
| pacs01 (Orthanc) | Application | Radiology | High |
| siem01 (Wazuh + Ansible) | Security | IT | High |
| dc02 file shares (departmental) | Data service | IT | Medium |
| portal01 (patient portal) | Application | Administration | Medium (internet-exposed) |
| cloud-dr + Blob backups | DR | IT | Medium |
| Clinician workstations (win-client sim.) | Endpoint | Clinical IT | Medium |
| Synthetic clinical records / public DICOM | Data | — | n/a (no real patient data) |

## 10. Updated Backup Matrix — Table 8.1 (3-2-1)

| System | Method | Frequency | Local copy | 2nd media | Off-site immutable | Retention |
|---|---|---|---|---|---|---|
| EMR database | MariaDB async replication → cloud-dr (binlog, RPO 15 min) **+** `mysqldump` → gzip → GPG | Continuous / nightly 01:00 | emr01 `/var/backups/emr` (7 d) | — | Azure Blob, immutability policy (365 d) | 90 d dumps |
| Domain controllers | `wbadmin` System State → `\\10.10.70.12\Backups` | Nightly | dc02 share | restic push from dc02 → Blob | Blob immutable | 30 d |
| File shares (dc02) | restic → Blob | Nightly 02:00 | dc02 local snapshot | — | Blob immutable | 30 d |
| PACS (pacs01) | restic of `/var/lib/orthanc` + re-downloadable public datasets | Nightly 03:00 | pacs01 local | — | Blob immutable | 30 d |
| pfSense config | XML config export (encrypted) | Nightly | siem01 | — | Blob immutable | 90 d |
| Wazuh/Ansible config | Git repo (GitHub private) + weekly archive | On change / weekly | siem01 | — | Blob immutable | 90 d |
| VMs (all) | OVA export | Monthly | `D:\NileCare\archive` | — | n/a (cold HDD) | 2 versions |

Immutable = Azure Blob **versioning + time-based immutability policy** (equivalent of S3 Object Lock — supports the ransomware scenarios).

## 11. RTO/RPO targets — Table 8.2 (unchanged, now covered)

| System | Criticality | Target RTO | Target RPO | Covered by |
|---|---|---|---|---|
| EMR/HIS | Critical | 1 hour | 15 minutes | cloud-dr replica (VPN) + immutable dumps |
| Active Directory | Critical | 30 min | 1 hour | Second DC + System State |
| PACS/imaging | High | 4 hours | 24 hours | restic + re-downloadable datasets |
| File server | Medium | 8 hours | 24 hours | restic |
| Patient portal | Low | 24 hours | 24 hours | rebuild via Ansible + config backup |

Measured values stay **placeholders** until Scenarios 7–8 are executed (measurement method in Deliverable C).

## 12. Updated Risk Register — Table 2.3 (R1–R6 preserved, R7–R13 added)

| ID | Risk | Likelihood | Impact | Mitigation | Chapter |
|---|---|---|---|---|---|
| R1 | Ransomware | High | Critical | Segmentation, immutable backups, FSRM screening | 3, 4, 7, 8 |
| R2 | Insider snooping on records | High | High | RBAC, audit logs, Wazuh alerts | 5, 7 |
| R3 | Unpatched medical devices | High | High | Isolated VLAN 30, apt-alias-only egress | 3, 7 |
| R4 | Power or hardware failure | Medium | Critical | Second DC, cloud DR | 4, 6 |
| R5 | Stolen credentials | Medium | High | MFA, lockout policy, monitoring | 4, 7 |
| R6 | Cloud or VPN outage | Low | Medium | Local backups remain primary | 6, 8 |
| **R7** | Lateral movement from user VLANs to servers | High | High | Dedicated VLAN 70 + inter-VLAN ACLs + host firewalls | 3, 7 |
| **R8** | Monitoring blind spot caused by segmentation | Medium | High | Limited 1514/1515 allows from every agent VLAN; daily agent check-in check | 3, 7 |
| **R9** | EMR data loss beyond 15-min RPO | Medium | Critical | MariaDB replication to cloud-dr + nightly GPG dump to immutable Blob | 5, 6, 8 |
| **R10** | Dual-role dc02 (DC + file server) blast radius | Medium | High | FSRM, hardened GPO, accepted in Limitations (host RAM) | 4, 11 |
| **R11** | Wrong Windows edition blocks AppLocker enforcement | Medium | Medium | DISM edition check before build; Enterprise Eval fallback | 4, 9 |
| **R12** | Cloud free-tier expiry / egress cost | Low | Medium | Local backups primary; lifecycle + cost alerts; D: cold archive | 6, 8 |
| **R13** | Unauthorised PACS access (Orthanc permissive defaults) | High | High | Orthanc auth, AET allowlist, VLAN 30 isolation, Wazuh FIM | 3, 5, 7 |

## 13. Corrections to template text (apply everywhere)

- §1.3 Objective 1 → "Deploy centralised identity, DNS, DHCP, and file services on **Windows Server 2025**."
- Table 3.3 OS column → Windows Server **2025** (dc01/dc02), Ubuntu Server **24.04 LTS** (emr01/pacs01/portal01/siem01/lin-client/cloud-dr), pfSense (fw01); retain only the selected Ubuntu and Wazuh stack; add pacs01 + portal01 rows.
- Ch5 §5.7 playbook row → "…UFW, Fail2ban, **AppArmor**".
- Declaration (page 2) → §14 below.
- Report says "180-day evaluation" for Server 2025 and recommends snapshots (stated in §1.4 Scope and §4.1).
- Limitations §11.3 → add: *"The win-client runs **Windows 11 Education unactivated** because AppLocker enforcement requires an Enterprise/Education edition: the team's downloaded Windows 11 25H2 media is the consumer ISO (Home/Education/Pro/Pro Education/Pro for Workstations — no Enterprise), and Education was selected during setup (index 4). Unactivated licensing shows a desktop watermark and restricts personalisation but does not expire and does not limit AppLocker; **a mandatory early test (B0 §2, step 9) must demonstrate real enforcement before any AppLocker evidence is accepted** — the documented fallback if it fails is Windows 11 Enterprise Evaluation (§15.2). The team's existing Windows 10 Pro VM (22H2, end-of-support since October 2025) is retained unmodified as a backup host and would only support AppLocker in audit mode. Windows Server 2025 runs under its 180-day evaluation. The pfSense CE media is the Netgate Installer v1.2, whose SHA256 was verified against Netgate's official checksums file; the installed pfSense version is recorded after installation. Microsoft does not publish SHA256 values for the Windows Server 2025 Evaluation or Windows 11 consumer ISOs; their integrity is documented by the official download-page filenames and the hashes recorded in §15.1 (the WS2025 ISO additionally verified identical across two independent copies on this workstation)."*

## 14. Declaration paragraph (replaces Synthea-only sentence)

> We declare that this project is our own work, that all sources are cited, and that no real patient data was used at any stage. All clinical records were generated with Synthea, an open-source synthetic patient generator; all medical images were obtained from publicly available, fully anonymised DICOM sample datasets (the Orthanc sample dataset and The Cancer Imaging Archive). Any resemblance to real patients, staff, or facilities is coincidental.

## 15. Install-media checklist (status 2026-11-18)

### 15.1 Media table (real filenames, sizes, SHA256 — measured, none invented)

| File in `D:\NileCare\iso\` | Size (bytes) | SHA256 | Official verification |
|---|---|---|---|
| `ubuntu-24.04.3-live-server-amd64.iso` | 3,303,444,480 | `c3514bf0056180d09376462a7a1b4f213c1d6e8ea67fae5c25099c6fd3d8274b` | ✅ matches `https://releases.ubuntu.com/24.04.3/SHA256SUMS` |
| `netgate-installer-v1.2-RELEASE-amd64.iso.gz` | 342,781,760 | `184514FE7DF0D339362C1E33FA051C464577A450528759B343ADE894C7C57955` | ✅ matches `https://www.netgate.com/hubfs/pfSense-plus-installer-checksums.txt` (official; hashes computed against the compressed file) |
| `netgate-installer-v1.2-RELEASE-amd64.iso` (extracted, 7-Zip) | 1,059,239,936 | `F55DC289EEDA16C9698DB092C93B3F26A36FDACAADC4FAB67876530BD3AEAE96` | ✅ verified transitively via the .gz above |
| `26100.1742.240906-0331.ge_release_svc_refresh_SERVER_EVAL_x64FRE_en-us.iso` | 6,014,152,704 | `D0EF4502E350E3C6C53C15B1B3020D38A5DED011BF04998E950720AC8579B23D` | ⚠️ Microsoft publishes no hash for eval ISOs; provenance = official Evaluation Center filename + two independent copies on this workstation hash-identical (`D:\NileCare\iso` ≡ `D:\وندسسس`) |
| `Win10_22H2_English_x64v1.iso` (retail) | 6,140,975,104 | `A6F470CA6D331EB353B815C043E327A347F594F37FF525F17764738FE812852E` | ℹ️ no longer needed for win-client after decision (a) — kept as install media for the backup Win10 Pro VM if it ever needs repair |
| `Win11_25H2_English_x64.iso` (consumer, 25H2) | 7,736,125,440 | `D141F6030FED50F75E2B03E1EB2E53646C4B21E5386047CB860AF5223F102A32` | ⚠️ source = Microsoft consumer download page (**not** Evaluation Center); Microsoft publishes no hash for it — provenance = official page filename + hash recorded here; editions verified by the team via DISM (Home/Education/Pro/Pro Education/Pro for Workstations — **no Enterprise**; **Education = Index 4, re-verify with elevated DISM before setup**) |

### 15.2 win-client media — DONE (consumer 25H2 ISO, Education edition)

The team downloaded `Win11_25H2_English_x64.iso` (7,736,125,440 bytes, consumer download page) — DISM shows **no Enterprise edition**, so the final decision is **Windows 11 Education (Index 4), unactivated** (education enforces AppLocker; unactivated = watermark, no expiry). Build steps = B0 §2, including the **mandatory early AppLocker enforcement test**.

**Fallback (only if the early enforcement test fails while unactivated):** Windows 11 Enterprise Evaluation — exact steps:
1. Open `https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise`.
2. Select **ISO — Enterprise Evaluation**, 64-bit, English; complete the short form if shown.
3. Save the ISO to `D:\NileCare\iso\`, record filename + SHA256 in `E:\NileCare\evidence\` (the page usually publishes no hash), rebuild win-client per B0 §2 with this ISO, and note the 90-day expiry in the report.

### 15.3 win-client build (decision (a), Rev6) — full procedure = B0 §2

Build the fresh **Windows 11 Education (Index 4, unactivated)** VM on D: (2 vCPU / **4 GB** / 64 GB thin / EFI + Secure Boot + **TPM 2.0** or LabConfig bypass / NIC1 → VMnet10 / hostname `WIN-CLIENT01` / snapshot `baseline-before-lab` / **early AppLocker enforcement test = step 9**). The Win10 Pro VM at `E:\Tools\WIN10` stays **untouched as backup** (edition evidence recorded; no changes to its NICs, disk, or snapshots).

### 15.4 pfSense — media DONE & verified; install = B0 §3

`netgate-installer-v1.2-RELEASE-amd64.iso` boots fw01; it **fetches pfSense CE over the internet at install time** (docs.netgate.com/install/netinstaller.html: "The Netgate Installer image does not contain installation packages for pfSense software, it fetches them over the Internet"). The installed version is recorded after install (Status → Dashboard; expected 2.9.x) — **single placeholder used everywhere: "pfSense CE (version recorded after install; expected 2.9.x)"**.

### 15.5 Remaining checks

1. DISM edition check of the WS2025 ISO (for the dc builds):
   ```
   dism /Get-WimInfo /WimFile:D:\NileCare\iso\26100.1742.240906-0331.ge_release_svc_refresh_SERVER_EVAL_x64FRE_en-us.iso\sources\install.wim
   ```
2. Confirm the Azure for Students subscription + region/SKU pre-flight (`az vm list-skus --location uaenorth --size Standard_B1s -o table` — B6 §1).
3. Snapshot policy: one **clean-baseline** snapshot per VM after build + hardening (`baseline-before-lab` on win-client; `fw01-baseline`, `dc01-pre-promo`, `dc02-pre-promo`…); delete scenario snapshots after evidence capture; monthly compaction. Disk rule: E: ≥ 15 GB free at all times.

**Open items (non-blocking):** (a) download the Windows 11 Enterprise Evaluation ISO (§15.2) and report the filename; (b) run the fw01 install via the Netgate Installer (B0 §3) and report the installed pfSense version from the dashboard; (c) confirm the Azure for Students subscription is active.

## 16. Deliverable sequence

All nine local VMs are integrated on Mahmoud's laptop. The demonstration plan uses at most four local VMs at a time. `cloud-dr` runs on Azure during a live DR demonstration. See the [demo schedule](../team/07-INTEGRATION-PLAYBOOK.md#5-demo-power-schedule).

The [team plan](../team/00-TEAM-PLAN.md) records build ownership. Mahmed Shaker prepares `siem01` and Wazuh. Youssef Al-Sayed configures and runs Ansible on the same VM after Shaker hands over the base build.

| Step | Deliverable | Output location |
|---|---|---|
| A | Design package (this document) | `E:\NileCare\report\A-design-package.md` |
| B | Implementation runbooks Ch4–8 | `E:\NileCare\report\B-runbooks\` |
| C | Test plan, 9 scenarios and acceptance criteria | `E:\NileCare\report\C-test-plan.md` |
| D | Security policies (ISO 27001 A + PDPL 151/2020) | `E:\NileCare\report\D-policies.md` |
| E | Report text (Intro, Abstract, Conclusion, Limitations, Future Work) | `E:\NileCare\report\E-report-text.md` |
