# B0 — Preparation Runbook: VMware networks, pfSense (fw01), lab-wide prerequisites
**Chapter 3 implementation (report §3.1–3.4) — required before Chapters 4–8.**

Prerequisites from Deliverable A: Table 3.1 (VLANs), Table 3.2 (VMnets), Table 3.4/3.5 (matrix + ports), power-state policy. Evidence naming: `E:\NileCare\evidence\<name>.png` (see B-README mapping).

---

## 1. VMware Workstation networks (Virtual Network Editor)

Run VMware **as administrator** → *Edit → Virtual Network Editor → Change Settings*.

| VMnet | Type | Subnet IP | Subnet mask | DHCP | Host adapter |
|---|---|---|---|---|---|
| VMnet8 (existing) | NAT | default (192.168.x.0/24) | — | on | — |
| VMnet10 | Host-only | 10.10.10.0 | 255.255.255.0 | **off** | **disabled** |
| VMnet12 | Host-only | 10.10.20.0 | 255.255.255.0 | off | **disabled** |
| VMnet13 | Host-only | 10.10.30.0 | 255.255.255.0 | off | **disabled** |
| VMnet14 | Host-only | 10.10.40.0 | 255.255.255.0 | off | **disabled** |
| VMnet15 | Host-only | 10.10.50.0 | 255.255.255.0 | off | **disabled** |
| VMnet16 | Host-only | 10.10.60.0 | 255.255.255.0 | off | **enabled** (MGMT) |
| VMnet17 | Host-only | 10.10.70.0 | 255.255.255.0 | off | **disabled** |

VMware Workstation supports only **VMnet0–VMnet19** — VMnet20–70 do not exist (v1.0 of this runbook used them; remapped in v1.3). **VMnet11 is intentionally unused** (spare). IPs, subnets, rules and ports are unchanged.

- *Add* → select "Host-only", untick "Use local DHCP service to distribute IP address to VMs".
- **Host adapter exists only on VMnet16** (already configured on this host and verified Preferred/Up): `VMware Network Adapter VMnet16` → IPv4 **10.10.60.100 / 255.255.255.0**, no default gateway, no DNS, **plus a static route `10.10.0.0/16 via 10.10.60.1`** so the administrator reaches every VLAN through pfSense (verify: `route print -4` shows the 10.10.0.0 route; `ping 10.10.70.1` succeeds once fw01 is up).
- VMware *Preferences → Memory → "Allow some virtual machine memory to be swapped"* (required by the 10.5 GB concurrency cap).

Evidence: `network-editor-vmnets.png` (report §3.2 support).

## 2. win-client — **Windows 11 Education (Index 4), unactivated** (decision (a) final form, Rev6)

**Step 0 — edition checks: DONE, results recorded.**
- Existing VM `E:\Tools\WIN10`: `wmic os get Caption` → **"Microsoft Windows 10 Pro"** — Pro cannot *enforce* AppLocker (audit-mode only) → not usable as win-client.
- Downloaded media `D:\NileCare\iso\Win11_25H2_English_x64.iso` (7,736,125,440 bytes, SHA256 `D141F603…02A32` — A §15.1) is the **consumer 25H2 ISO**: DISM shows Home / Education / Pro / Pro Education / Pro for Workstations (+N variants) — **no Enterprise**.
- **Decision (a) final form:** win-client = **Windows 11 Education** (Education enforces AppLocker), running **unactivated** (watermark + personalisation limits only; **no expiry** — friendlier than the 90-day Enterprise Eval). The Win10 Pro VM stays **untouched as backup** — do NOT delete "Snapshot 1", do NOT change its NICs, disk, or hostname.

### 2.1 Build the new VM (one small step at a time)

1. **Verify the edition index (elevated PowerShell/CMD — once, before setup):**
   ```
   REM Mount the ISO in Windows first. Replace X: with the mounted drive letter.
   dism /Get-WimInfo /WimFile:X:\sources\install.wim
   ```
   Confirm **Index 4 = Windows 11 Education** (index order can differ per media — if it differs, use the Education index number you see).
2. **Create the VM** (VMware *File → New Virtual Machine → Custom*): name `win-client`, location **`D:\NileCare\vms\win-client`** (D: HDD, cold set); **2 vCPU**, **4096 MB RAM**, **64 GB** disk, firmware **EFI + Secure Boot**.
3. **TPM 2.0 — primary path:** VM Settings → *Options → Access Control → Encrypt* → choose **"Encrypt only the files needed to support a TPM"** (password → vault) → *Hardware → Add → Trusted Platform Module*. With vTPM + Secure Boot + 4 GB RAM the Win11 setup hardware checks pass **without any bypass**.
   **Fallback path (only if TPM/encryption fails in your Workstation build):** boot the ISO, at the first setup screen press **Shift+F10** → in the Command Prompt run:
   ```
   reg add HKLM\System\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1 /f
   reg add HKLM\System\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
   reg add HKLM\System\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1 /f
   ```
   → close, click *Install Now* again (LabConfig bypass is a documented Microsoft-leaked check skip; note in the report if used).
4. **Network:** one NIC only → **VMnet10** (VLAN 10). No second NIC.
5. **Install Windows:** boot the ISO → **select the edition "Windows 11 Education"** (the edition picker appears on consumer media; verify it matches the index you confirmed in step 1) → complete setup (local account where offered).
6. **Post-install:** install **VMware Tools**; run Windows Update once; `powercfg /h off`; optionally cap pagefile to 2 GB. Expect the **"Activate Windows" watermark** (unactivated — normal, documented).
7. **Rename the host:** `Rename-Computer -NewName WIN-CLIENT01 -Restart`.
8. **Record the MAC — mandatory placeholder:** VM Settings → *Network Adapter → Advanced* → copy the MAC → write it into **B4 §3's `<VERIFY-MAC-AFTER-NIC-SWAP>`** (do not reuse the backup VM MAC).
9. **MANDATORY EARLY TEST — AppLocker must actually enforce while unactivated** (before the baseline snapshot and before domain join):
   ```powershell
   # save as C:\Tests\applocker-test.xml — default-allow Windows/Program Files, DENY the test folder
   New-Item -ItemType Directory -Path C:\Tests -Force | Out-Null
   Copy-Item C:\Windows\System32\notepad.exe C:\Tests\blockme.exe
   @'
   <AppLockerPolicy Version="1">
     <RuleCollection Type="Exe" EnforcementMode="Enabled">
       <FilePathRule Id="db507acf-67ef-4b98-a29b-72202ba04974" Name="Allow Windows" Description="Allow the Windows directory" UserOrGroupSid="S-1-1-0" Action="Allow">
         <Conditions><FilePathCondition Path="C:\Windows\*"/></Conditions></FilePathRule>
       <FilePathRule Id="70da52df-1bbf-40fd-a330-91a8b56e2e89" Name="Allow Program Files" Description="Allow the Program Files directory" UserOrGroupSid="S-1-1-0" Action="Allow">
         <Conditions><FilePathCondition Path="C:\Program Files\*"/></Conditions></FilePathRule>
       <FilePathRule Id="f47a2043-6903-497b-a428-28646d0761f1" Name="Deny test folder" Description="Deny executables in the test folder" UserOrGroupSid="S-1-1-0" Action="Deny">
         <Conditions><FilePathCondition Path="C:\Tests\*"/></Conditions></FilePathRule>
     </RuleCollection>
   </AppLockerPolicy>
   '@ | Set-Content C:\Tests\applocker-test.xml
   Set-AppLockerPolicy -XmlPolicy C:\Tests\applocker-test.xml -ErrorAction Stop
   Get-Service AppIDSvc | Start-Service -ErrorAction Stop
   C:\Tests\blockme.exe      # expected: "This app has been blocked by your system administrator"
   ```
   **PASS** = the exe is blocked (screenshot `applocker-early-test.png` into `E:\NileCare\evidence\`) → Education enforces while unactivated → continue to step 10. **FAIL** = it runs despite verified policy import and a running Application Identity service → investigate enforcement before concluding that the edition is the cause → **fall back to Windows 11 Enterprise Evaluation** (download + rebuild per A §15.2 fallback block) and tell me immediately.
10. **Reset the test policy** (so the domain GPO applies cleanly later): `Set-AppLockerPolicy -XmlPolicy` with an empty Exe collection or delete via *Security Settings → Application Control Policies* reset — then shut down → *Take Snapshot* → name **`baseline-before-lab`** (no memory). Restoring it returns the "ready to domain-join, pre-B4" state.
11. Boot → once B4 §3 is done the machine receives `10.10.10.110` via the DHCP relay, then joins the domain (B4 §9).

RAM note: win-client runs at **4 GB** (Win11 minimum). Use the phase schedules in A §5 and Deliverable C.

## 3. fw01 — pfSense CE install **via the Netgate Installer v1.2**

**Media (verified):** `D:\NileCare\iso\netgate-installer-v1.2-RELEASE-amd64.iso` — extracted from `netgate-installer-v1.2-RELEASE-amd64.iso.gz` whose SHA256 (`184514FE…57955`) **matches Netgate's official checksums file** (`https://www.netgate.com/hubfs/pfSense-plus-installer-checksums.txt`; hashes are computed against the compressed file, so the extracted ISO is verified transitively).
**What this installer is (official docs, docs.netgate.com/install/netinstaller.html):** it bundles **no packages** — at install time it **fetches pfSense CE or Plus over the internet**; a system not entitled to Plus gets **pfSense CE**. Therefore fw01's **WAN NIC (VMnet8 NAT) must have internet during the install**.
**Honesty note:** the installer screens have **not been verified** by this runbook (the author has not seen them) — steps below describe intent, not exact button labels. The installed pfSense **version is recorded after install** (Status → Dashboard; expected 2.9.x) — until then every reference says: **"pfSense CE (version recorded after install; expected 2.9.x)"**.

1. Create VM (E:\NileCare\vms\fw01, E: SSD): 1 vCPU, **1024 MB**, 16 GB thin, EFI off (BIOS), **8 NICs** in this order — **first NIC MUST be WAN**: VMnet8 (WAN), VMnet10 (Clinical), VMnet12 (Admin), VMnet13 (Medical), VMnet14 (DMZ), VMnet15 (Guest), VMnet16 (MGMT), VMnet17 (Servers).
   **Platform check:** VMware Workstation Pro supports up to **10 virtual network adapters per VM** (VMware/Broadcom TechDocs, "Configuring Virtual Network Adapter Settings") — 8 NICs is within the supported limit; cite this in report §3.3.
2. **Boot the installer:** VM Settings → CD/DVD → *Use ISO image* → `D:\NileCare\iso\netgate-installer-v1.2-RELEASE-amd64.iso` → power on.
3. **Expect (unverified screens):** a boot menu → the installer starts → it needs network on the WAN NIC (VMnet8 NAT provides DHCP internet) → it fetches the current pfSense CE packages → select the target disk (the 16 GB disk — **double-check it is NOT your host disk**; in the VM only the virtual disk is visible) → accept → install → power off when done.
4. **After install:** VM Settings → CD/DVD → **disconnect the installer ISO** → power on → the pfSense console appears.
5. Console: **No** to VLANs (we use physical NICs) → assign: `em0`=WAN, then `em1..em7` = OPT1..OPT7 → set interface IPs:

| Interface | NIC | Name in pfSense | IP |
|---|---|---|---|
| em0 | VMnet8 | WAN | DHCP (from NAT) |
| em1 | VMnet10 | CLINICAL | 10.10.10.1/24 |
| em2 | VMnet12 | ADMIN | 10.10.20.1/24 |
| em3 | VMnet13 | MEDICAL | 10.10.30.1/24 |
| em4 | VMnet14 | DMZ | 10.10.40.1/24 |
| em5 | VMnet15 | GUEST | 10.10.50.1/24 |
| em6 | VMnet16 | MGMT | 10.10.60.1/24 |
| em7 | VMnet17 | SERVERS | 10.10.70.1/24 |

4. Console option 2 per OPT interface: IP, mask 24, DHCP **off**, "enable interface as DHCP server: no". Set LAN→MGMT as webGUI interface. Enable SSH (option 14) restricted to MGMT later.
5. WebGUI from host: `https://10.10.60.1` (factory credentials on the console banner — change immediately in step 6) → **record the installed pfSense version: Status → Dashboard (or console option 8: `cat /etc/version`)** → fill the placeholder **"pfSense CE (version recorded after install; expected 2.9.x)"** in A §15.1/§8, B-README and the report → System → General Setup: hostname `fw01`, domain `nilecare.local`, DNS servers 1.1.1.1 + 9.9.9.9 (use gateway WAN).
6. **Disable factory credentials**: System → User Manager → change `admin` password (vault) → System → Advanced → Admin Access: protocol HTTPS, SSH only from 10.10.60.0/24.
7. Take snapshot `fw01-baseline` (power off first).

**Fallback if the Netgate Installer fails in VMware** (no internet on WAN, screen hangs, or disk not detected):
1. Check the official docs chapter *Installing and Upgrading* (docs.netgate.com/pfsense/en/latest/install/ — pages "Netgate Installer", "Download Installation Media", "Perform the Installation", "Assign Interfaces").
2. If the store still offers the classic CE image (search **"pfSense CE"** on shop.netgate.com), download `pfSense-CE-<version>-RELEASE-amd64.iso.gz` + its `.sha256`, verify with `Get-FileHash` **before decompressing**, extract with 7-Zip, and follow the classic installer path in this §3 (boot ISO → Auto (ZFS) → install → remove ISO) — then continue at step 4 above.
3. Common installer hiccups: WAN has no internet → confirm VMnet8 NAT gives DHCP (test with any other VM on VMnet8); installer boots to UEFI shell → set firmware type to BIOS (or vice versa); disk list empty → recreate the VM disk as NVMe/SATA per the docs' virtualization chapter.

Evidence: `fw01-interface-assignments.png`.

## 4. pfSense services

### 4.1 DHCP Relay (design fix #1) — `Services → DHCP Relay`
- Disable the pfSense DHCP server on all interfaces. Keep the relay off during Gate 1.
- In Gate 2, create the scopes in B4 §3 before enabling relay on **CLINICAL, ADMIN, GUEST** to **10.10.70.11**. Leave circuit ID off.
- Allow DHCP broadcasts from Guest clients (UDP 68 → 67, including source 0.0.0.0 and destination 255.255.255.255) to be handled locally by the pfSense relay. Verify the relay-generated packets reach dc01. Keep the Guest-to-Servers deny in place. Do not add a routed Guest-to-dc01 allow.

### 4.2 Guest addressing and isolation
- Gate 1: use a test client on VMnet15 with **10.10.50.60/24**, gateway/DNS **10.10.50.1**.
- Gate 2: return the test client to DHCP after the Guest scope and relay are active. Verify a lease, DNS/HTTPS and the logged Guest-to-Servers deny. Scenario 4 uses the controlled static test IP.

### 4.3 DNS Resolver (unbound) — `Services → DNS Resolver`
- Enable; listen on all interfaces; **enable "DNS Resolution: Forwarder"**? No — keep Resolver mode with upstream forwarders: General → DNS Servers (step 3) + tick "DNS Query Forwarding: Use DNS Servers specified…".
- *Domain Overrides*: domain `nilecare.local` → IP `10.10.70.11` (non-domain segments resolve AD names via the DC).
- *Host Overrides* (for resilience if DC is down): `emr01.nilecare.local → 10.10.70.21`, `pacs01.nilecare.local → 10.10.30.21`, `portal01.nilecare.local → 10.10.40.10`.

### 4.4 NTP — `Services → NTP`
- Interfaces: **MEDICAL, DMZ, GUEST**; upstream `pool.ntp.org`. Servers of VLAN 30/40/50 use `10.10.x.1` (chrony config in B5).

### 4.5 Aliases — `Firewall → Aliases`

| Alias | Type | Entries |
|---|---|---|
| DCS | Host(s) | 10.10.70.11, 10.10.70.12 |
| EMR01 | Host(s) | 10.10.70.21 |
| PACS01 | Host(s) | 10.10.30.21 |
| PORTAL01 | Host(s) | 10.10.40.10 |
| SIEM01 | Host(s) | 10.10.60.21 |
| CLOUD_DR | Host(s) | 172.16.10.10 |
| AD_PORTS | Port(s) | 53, 88, 464, 389, 636, 3268, 445, 135, 49152-49252, 123 |
| AGENT_PORTS | Port(s) | 1514, 1515 |
| SYSLOG_PORT | Port(s) | 514 |
| NON_INTERNET | Network(s) | RFC1918 (10/8, 172.16/12, 192.168/16), 100.64.0.0/10, 127.0.0.0/8, 169.254.0.0/16 and any additional local or VPN IPv4 ranges |
| UBUNTU_ARCHIVE | Host(s), DNS names | archive.ubuntu.com, security.ubuntu.com, changelogs.ubuntu.com, azure.archive.ubuntu.com, ports.ubuntu.com |
| MGMT_HOSTS | Host(s) | 10.10.60.21, 10.10.60.100 |

UBUNTU_ARCHIVE is a DNS-host alias, not a URL-table alias: those names are hostnames, not feeds of CIDR addresses. Keep the local/lab deny ahead of its egress allow. For each table below, add local-service exceptions first, then a logged block to This Firewall (self), then public egress. Use separate rules where a row names multiple destinations. Guest DHCP is an additional local service in Gate 2. MGMT-to-Guest remains blocked before all management allows.

### 4.6 WAN port-forward — `Firewall → NAT → Port Forward`
- WAN TCP 443 (any source) → **PORTAL01 443**, log on. (Public patient portal — the only inbound rule.)

## 5. Firewall ruleset (per-interface inbound; implements Table 3.4 + 3.5)

Create rules **top-down** on each interface; each interface ends with an explicit **Block + Log** rule for remaining traffic. Replies to established flows are automatic. Use IPv4 for this ruleset. For public egress use destination **NON_INTERNET with Invert match selected**, preceded by a logged block to **This Firewall (self)** after explicit local-service exceptions. Existing sessions may need their matching states cleared before an isolation retest.

**CLINICAL interface (inbound from 10.10.10.0/24)**

| # | Action | Proto | Source | Destination | Port | Log | Purpose |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | Clinical net | DCS | AD_PORTS | no | AD auth, DNS, NTP |
| 2 | Pass | TCP | Clinical net | EMR01 | 443 | no | OpenEMR |
| 3 | Pass | TCP | Clinical net | PACS01 | 443 | no | DICOMweb |
| 4 | Pass | TCP | Clinical net | PORTAL01 | 443 | no | portal testing |
| 5 | Pass | TCP | Clinical net | SIEM01 | AGENT_PORTS | no | Wazuh agent |
| 6 | Pass (disabled until cutover) | TCP | Clinical net | CLOUD_DR | 443 | yes | DR cutover path, B8 §4 |
| 7 | Pass | TCP | Clinical net | !NON_INTERNET | 80,443 | no | public web egress |
| 8 | **Block** | any | Clinical net | any | any | **yes** | default deny, logged |

**ADMIN interface**: same rows 1–8 plus row 1.5 `Pass TCP Admin→DCS 445 (shares)` (covered by AD_PORTS) and drive-mapping SMB. PACS access is TCP 443 to PACS01 only, like Clinical. Otherwise identical. Log the block row.

**MEDICAL interface**

| # | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | Medical net | SIEM01 | AGENT_PORTS | no |
| 1a | Pass | TCP | PACS01 | SIEM01 | 514 | no |
| 1b | Pass | TCP/UDP | Medical net | This Firewall (self) | 53 | no |
| 1c | Pass | UDP | Medical net | This Firewall (self) | 123 | no |
| 1d | Block | any | Medical net | NON_INTERNET or This Firewall (self), as separate rules | any | yes |
| 2 | Pass | TCP | Medical net | UBUNTU_ARCHIVE | 80,443 | no |
| 3 | **Block** | any | Medical net | any | any | **yes** |

**DMZ interface**: allow TCP 1514/1515 to SIEM01 for agents and TCP 514 to SIEM01 from PORTAL01. Allow TCP 443 from PORTAL01 to EMR01. Allow the required local DNS/NTP, block local and lab destinations after the specific exceptions, then allow TCP 80/443 to UBUNTU_ARCHIVE. End with a logged block.

**GUEST interface**

| # | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | Guest net | This firewall (self) | 53 | no |
| 2 | Pass | TCP | Guest net | !NON_INTERNET | 80,443 | no |
| 3 | **Block** | any | any | any | any | **yes** |

**MGMT interface**

| # | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 0 | **Block** | any | MGMT net | Guest net | any | **yes** |
| 1 | Pass | TCP | MGMT net | any internal | 22 | no | SSH/Ansible (whole /24; host-level protection remains enabled on Linux targets) |
| 2 | Pass | TCP | MGMT_HOSTS | Windows net/clients | 3389, 5985 | no | RDP/WinRM |
| 3 | Pass | TCP | MGMT_HOSTS | any internal | 443 | no | web UIs |
| 4 | Pass | TCP/UDP | MGMT net | any | 53,123 | no | DNS/NTP |
| 5 | Pass | TCP | MGMT net | !NON_INTERNET | 80,443 | no | public updates |
| 6 | **Block** | any | MGMT net | any | any | **yes** |

**SERVERS interface**

| # | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | SERVERS net | SIEM01 | AGENT_PORTS | no |
| 1a | Pass | TCP | EMR01 | SIEM01 | 514 | no |
| 2 | Pass | TCP | SERVERS net | !NON_INTERNET | 80,443 | no | public updates and backup endpoints |
| 2a | Pass | UDP | SERVERS net | !NON_INTERNET | 123 | no | upstream NTP |
| 3 | **Block** | any | SERVERS net | any | any | **yes** |

**IPsec interface**, when B6 is implemented: allow TCP 3306 only from CLOUD_DR to EMR01. Return traffic uses the established state. B6 also defines cloud-dr agent telemetry.

**WAN interface**: default deny (log packets matched from default block rules: enable in System → Advanced → Firewall & NAT); no inbound passes except the port-forward from 4.6.

- `Status → System Logs → Settings → Remote Logging`: remote log server **10.10.60.21:514**, all firewall events (suricata in B7).

Evidence: `fig-7.1-firewall-rules-per-interface.png` (screenshot each interface rules screen).

## 6. Suricata IDS (detail/tuning in B7) — `Services → Suricata`
- Install package; enable on **WAN, CLINICAL, MEDICAL, DMZ, SERVERS, GUEST** in **IDS mode** (no inline blocks in the lab); rules: ET open + suricata.net; *Interface Settings → Logging*: enable syslog alerts; Global: update rules every 12 h, suppress SID noise (start with default suppressions; tune in B7.3).

## 7. OpenVPN remote access with MFA (report §7.2)
1. `System → User Manager`: create `clinician1` → password + **OTP seed** (TOTP secret, 6 digits, 30 s) — import seed into an authenticator app.
2. `VPN → OpenVPN → Servers` (wizard): Remote Access (User Auth — Yes, TLS Auth), protocol UDP, port **1194**, tunnel network **10.10.90.0/24**, redirect gateway off; local networks: `10.10.70.0/24`, `10.10.10.0/24`; DNS default 10.10.70.11; Duplicate CN off; Topology subnet.
3. Firewall: WAN pass **UDP 1194 → this firewall** (log on); OPENVPN interface: pass UDP/TCP 53 → DCS, pass TCP 443 → EMR01, block+log rest (remote clinicians = clinical ruleset, Table 3.5 row 19).
4. Export user config (OpenVPN Client Export utility package) for clinician1.
5. Honest test note: pfSense WAN is NAT'd behind the home router — inbound testing from the internet needs a home-router port-forward of 1194/UDP; test from the host's home network or mark "tested from LAN only" in the report.

## 8. Acceptance checks (record outputs as evidence)

1. Verify Virtual Network Editor settings against §1 and save `network-editor-vmnets.png`.
2. On the MGMT host, check `curl.exe -k -I https://10.10.60.1` and capture the pfSense interface assignments.
3. Use the static Guest test client from §4.2. Verify DNS through 10.10.50.1 and HTTPS to a public site. Attempt one connection to 10.10.70.11 TCP 443 and verify the matching logged deny in pfSense. A timeout alone is insufficient because dc01 is not deployed yet.
4. Record the results in `gates.log` and save the deny as `fw-default-deny-log.png`. Verify DHCP leases in Gate 2.

## Screenshots to capture (evidence\)

| File | Content | Report figure |
|---|---|---|
| fig-3.1-network-diagram.png | draw.io diagram built from A §2–§5 (VLANs, fw01 8 NICs, VPN, cloud) | Figure 3.1 |
| fw01-interface-assignments.png | pfSense Interfaces overview | §3.2 |
| fw01-dhcp-relay.png | DHCP Relay page | §4.2 |
| fig-7.1-firewall-rules-per-interface.png | rules per interface (Ch7 doubles as template fig) | Figure 7.1 |
| fw-default-deny-log.png | logged blocks | §3.4 |

**Power off fw01 only when no lab runs** — it is always-on. Next: `B4-windows.md`.
