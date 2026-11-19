# B4 — Windows Server Implementation Runbook (report Chapter 4)
Hosts: **dc01** (always on), **dc02** (power on for §6–§8), **win-client** (power on for §9). Domain `nilecare.local`.

---

## 1. dc01 — installation and forest promotion (§4.1)

1. Create VM (E:\NileCare\vms\dc01, E: SSD): 2 vCPU, **2560 MB**, 24 GB thin, 1 NIC → **VMnet17** (VLAN 70). Mount `D:\NileCare\iso\26100.1742.240906-0331.ge_release_svc_refresh_SERVER_EVAL_x64FRE_en-us.iso` (SHA256-verified, A §15.1). Install **Windows Server 2025 Desktop Experience** (Standard, evaluation — 180 days; record build with `winver`).
2. Post-install: rename `dc01`, set Administrator password, install VMware Tools, snapshot **`dc01-pre-promo`**.
3. Networking (GUI or PowerShell):
   ```powershell
   New-NetIPAddress -InterfaceAlias Ethernet0 -IPAddress 10.10.70.11 -PrefixLength 24 -DefaultGateway 10.10.70.1
   Set-DnsClientServerAddress -InterfaceAlias Ethernet0 -ServerAddresses 10.10.70.11
   Rename-Computer -NewName dc01 -Restart
   ```
4. Promote:
   ```powershell
   Install-WindowsFeature AD-DomainServices, DNS -IncludeManagementTools
   Install-ADDSForest -DomainName nilecare.local -DomainNetbiosName NILECARE `
     -DomainMode Win2025 -ForestMode Win2025 `
     -InstallDns:$true -SafeModeAdministratorPassword (Read-Host -AsSecureString) -Force
   ```
5. Post-promo:
   ```powershell
   Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target nilecare.local
   w32tm /config /manualpeerlist:"pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
   Restart-Service w32time
   dcdiag /s:dc01 /v > E:\NileCare\evidence\dcdiag-dc01.txt   # or run on dc01, copy out
   ```
6. Time-source note: dc01 (PDC) is authoritative NTP for the AD domain; VLANs 30/40/50 use pfSense (B0 §4.4).

## 2. DNS (§4.2)
1. Reverse zones for all subnets (DNS Manager → Reverse Lookup Zones → New): seven zones `10.10.10.0/24` through `10.10.70.0/24` (`10.10.x.in-addr.arpa`), allow dynamic updates "Secure only".
2. Forwarders: DNS Manager → dc01 → Properties → Forwarders: **1.1.1.1**, **9.9.9.9**.
3. Enable DNS scavenging (Properties → Advanced → Enable automatic scavenging, 7 days).
4. Verify: `Resolve-DnsName dc01.nilecare.local`, `nslookup google.com` from dc01.

## 3. DHCP scopes (§4.2) — pfSense relays VLAN 10/20 to dc01 (B0 §4.1)
```powershell
Add-WindowsFeature DHCP -IncludeManagementTools
Netsh dhcp add securitygroups; Restart-Service dhcpserver
# Authorize in AD:
Add-DhcpServerInDC -DnsName dc01.nilecare.local -IPAddress 10.10.70.11

Add-DhcpServerv4Scope -Name "VLAN10-Clinical" -StartRange 10.10.10.100 -EndRange 10.10.10.200 `
  -SubnetMask 255.255.255.0 -State Active -LeaseDuration 8:00:00
Set-DhcpServerv4OptionValue -ScopeId 10.10.10.0 -Router 10.10.10.1 -DnsServer 10.10.70.11,10.10.70.12 `
  -DnsDomain nilecare.local

Add-DhcpServerv4Scope -Name "VLAN20-Admin" -StartRange 10.10.20.100 -EndRange 10.10.20.200 `
  -SubnetMask 255.255.255.0 -State Active -LeaseDuration 8:00:00
Set-DhcpServerv4OptionValue -ScopeId 10.10.20.0 -Router 10.10.20.1 -DnsServer 10.10.70.11,10.10.70.12 `
  -DnsDomain nilecare.local

# Guest scope for Gate 2. Give guests only the local gateway/resolver.
Add-DhcpServerv4Scope -Name "VLAN50-Guest" -StartRange 10.10.50.50 -EndRange 10.10.50.250 -SubnetMask 255.255.255.0 -State Active -LeaseDuration 02:00:00
Set-DhcpServerv4OptionValue -ScopeId 10.10.50.0 -Router 10.10.50.1 -DnsServer 10.10.50.1
Add-DhcpServerv4ExclusionRange -ScopeId 10.10.50.0 -StartRange 10.10.50.60 -EndRange 10.10.50.60
# Reservation for the clinical test workstation — MAC is a PLACEHOLDER:
# record the real MAC from VMware (win-client → VM Settings → Network Adapter → Advanced) after the
# NIC is configured on VMnet10 (B0 §2 step 8). Never reuse the backup VM MAC.
Add-DhcpServerv4Reservation -ScopeId 10.10.10.0 -IPAddress 10.10.10.110 -ClientId <VERIFY-MAC-AFTER-NIC-SWAP>
```
**No scope for VLAN 70** (servers static). Activate the relay per B0 §4. Verify the Guest lease and logged isolation. Evidence: DHCP console with all three scopes + relay screenshot → `fig-4.2-dhcp-scopes.png`; `fw01-dhcp-relay.png` (B0).

## 4. OUs, users, groups (§4.3)

Run on dc01 as Administrator. OU design (matches template Table 4.x + A §4):

```
DC=nilecare,DC=local
 └ OU=NileCare
    ├ OU=Clinical   └ OU=Users | OU=Computers
    ├ OU=Admin      └ OU=Users | OU=Computers
    ├ OU=Medical    └ OU=Computers        (empty — rationale in report §3.2)
    ├ OU=Servers    └ OU=Computers
    ├ OU=Groups
    └ OU=ServiceAccounts
```

```powershell
# --- create-structure.ps1 (full script, E:\NileCare\repo\scripts\create-structure.ps1) ---
$paths = @(
  "OU=Clinical,OU=NileCare,DC=nilecare,DC=local",
  "OU=Admin,OU=NileCare,DC=nilecare,DC=local",
  "OU=Medical,OU=NileCare,DC=nilecare,DC=local",
  "OU=Servers,OU=NileCare,DC=nilecare,DC=local",
  "OU=Groups,OU=NileCare,DC=nilecare,DC=local",
  "OU=ServiceAccounts,OU=NileCare,DC=nilecare,DC=local"
)
foreach ($p in $paths) { New-ADOrganizationalUnit -Name $p.Split(',')[0].Substring(3) -Path ($p -replace "^[^,]+,","") }
New-ADOrganizationalUnit -Name Users -Path "OU=Clinical,OU=NileCare,DC=nilecare,DC=local"
New-ADOrganizationalUnit -Name Computers -Path "OU=Clinical,OU=NileCare,DC=nilecare,DC=local"
New-ADOrganizationalUnit -Name Users -Path "OU=Admin,OU=NileCare,DC=nilecare,DC=local"
New-ADOrganizationalUnit -Name Computers -Path "OU=Admin,OU=NileCare,DC=nilecare,DC=local"
New-ADOrganizationalUnit -Name Computers -Path "OU=Servers,OU=NileCare,DC=nilecare,DC=local"

# --- create-groups.ps1 ---
$g = @{
  "GRP-Doctors-ER"  = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
  "GRP-Nurses"      = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
  "GRP-Pharmacists" = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
  "GRP-Billing"     = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
  "GRP-Radiology"   = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
  "GRP-IT-Admins"   = "OU=Groups,OU=NileCare,DC=nilecare,DC=local"
}
foreach ($name in $g.Keys) { New-ADGroup -Name $name -GroupScope Global -GroupCategory Security -Path $g[$name] }

# --- create-users.ps1 (synthetic staff; password set once, must change at first logon) ---
$users = @(
  @{F="Omar";      L="Hassan";    Dept="Clinical"; Grp="GRP-Doctors-ER";  Title="ER Physician";   Vip=$true},
  @{F="Fatma";     L="Ibrahim";   Dept="Clinical"; Grp="GRP-Doctors-ER";  Title="ER Physician"},
  @{F="Mariam";    L="Tarek";     Dept="Clinical"; Grp="GRP-Nurses";      Title="Nurse"},
  @{F="Youssef";   L="Adel";      Dept="Clinical"; Grp="GRP-Nurses";      Title="Nurse"},
  @{F="Salma";     L="Fahmy";     Dept="Clinical"; Grp="GRP-Pharmacists"; Title="Pharmacist"},
  @{F="Khaled";    L="Mansour";   Dept="Clinical"; Grp="GRP-Radiology";   Title="Radiologist"},
  @{F="Hoda";      L="Zaki";      Dept="Admin";    Grp="GRP-Billing";     Title="Billing Clerk"},
  @{F="Ahmed";     L="Samir";     Dept="Admin";    Grp="GRP-Billing";     Title="Accountant"},
  @{F="Nada";      L="Rashad";    Dept="Admin";    Grp="GRP-Billing";     Title="Receptionist"}
)
$ou = @{ Clinical="OU=Users,OU=Clinical,OU=NileCare,DC=nilecare,DC=local";
         Admin="OU=Users,OU=Admin,OU=NileCare,DC=nilecare,DC=local" }
foreach ($u in $users) {
  $sam = "$($u.F.ToLower()).$($u.L.ToLower())"
  # Password: generate a RANDOM value per user in your password vault, paste below, and
  # ChangePasswordAtLogon forces replacement at first sign-in. Never commit the vault value.
  New-ADUser -Name "$($u.F) $($u.L)" -SamAccountName $sam -GivenName $u.F -Surname $u.L `
    -DisplayName "$($u.F) $($u.L)" -Title $u.Title -Department $u.Dept `
    -Path $ou[$u.Dept] -AccountPassword (ConvertTo-SecureString "<SET-IN-VAULT>" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true -Enabled $true
  Add-ADGroupMember -Identity $u.Grp -Members $sam
}
# VIP patient test pairing: user with extra group for scenario 6
Add-ADGroupMember -Identity "GRP-Doctors-ER" -Members "omar.hassan"

# --- admin accounts (least privilege: separate from daily accounts) ---
New-ADUser -Name "adm.it1" -SamAccountName "adm.it1" -Path "OU=ServiceAccounts,OU=NileCare,DC=nilecare,DC=local" `
  -AccountPassword (ConvertTo-SecureString "<SET-IN-VAULT>" -AsPlainText -Force) -Enabled $true
Add-ADGroupMember "Domain Admins" adm.it1

# --- break-glass (disabled; monitored; enable only in DR) ---
New-ADUser -Name "svc-breakglass" -SamAccountName "svc-breakglass" `
  -Path "OU=ServiceAccounts,OU=NileCare,DC=nilecare,DC=local" `
  -AccountPassword (ConvertTo-SecureString "<SET-IN-VAULT>" -AsPlainText -Force) -Enabled $false -Description "Break-glass DR account"
```
Evidence: `fig-4.1-aduc-ou-structure.png` (ADUC showing the tree + groups).

## 5. Group Policy (§4.4) — run on dc01 (GPMC)

| GPO | Link | Key settings (exact path) |
|---|---|---|
| Security-Baseline | OU=NileCare | Computer→Policies→Windows Settings→Security Settings→Account Policies: password 12+ chars, complexity on, history 12, max age 90 d; lockout 5 attempts / 15 min; Local Policies→Security Options: "Network security: Restrict NTLM: NTLMv2 only", "SMB signing always" |
| Lock-Screen-5min | Clinical, Admin | User→Admin Templates→Control Panel→Personalization: Screen saver timeout 300 s, password protect on |
| Block-USB | Clinical, Admin | Computer→Admin Templates→System→Removable Storage Access: Deny read/write all classes |
| Drive-Mapping-<Dept> | per dept OU | User→Preferences→Windows Settings→Drive Maps: `\\dc02\<dept>$` → H: |
| AppLocker | Clinical, Admin | see §5.1 — **enforced** on the Windows 11 **Education** win-client (unactivated, decision (a) Rev6); the backup Win10 Pro VM supports audit-mode only |
| Windows-Firewall | OU=NileCare | Computer→Security Settings→Firewall: Domain profile ON, inbound default Block, log `%systemroot%\system32\logfiles\firewall\pfirewall.log` |
| RDP-Restrict | Servers | Computer→Security Settings→Local Policies→User Rights: "Allow log on through Remote Desktop Services" = GRP-IT-Admins only |
| RPC-Restrict | Servers | Registry policy: `HKLM\SOFTWARE\Microsoft\Rpc\Internet` → `Ports` (REG_MULTI_SZ) = `49152-49252`, `PortsInternetAvailable` (REG_SZ) = `N`, `UseInternetPorts` (REG_SZ) = `N` (design fix: documented restricted range matching pfSense AD_PORTS) |
| Update-Deferral | Servers / Clinical | Computer→Admin Templates→Windows Components→Windows Update→Manage end-user experience: quality-update deferral 3 d (Servers) / 7 d (Clinical) — staged rollout substitute for WSUS (see §8) |

```powershell
$gpos = "Security-Baseline","Lock-Screen-5min","Block-USB","Drive-Mapping-Dept","AppLocker","Windows-Firewall","RDP-Restrict","RPC-Restrict","Update-Deferral"
foreach ($g in $gpos) { New-GPO -Name $g }
New-GPLink -Name "Security-Baseline" -Target "OU=NileCare,DC=nilecare,DC=local" -LinkEnabled Yes -Order 1
New-GPLink -Name "Windows-Firewall" -Target "OU=NileCare,DC=nilecare,DC=local" -LinkEnabled Yes -Order 2
New-GPLink -Name "Lock-Screen-5min" -Target "OU=Clinical,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "Lock-Screen-5min" -Target "OU=Admin,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "Block-USB" -Target "OU=Clinical,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "Block-USB" -Target "OU=Admin,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "AppLocker" -Target "OU=Clinical,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "AppLocker" -Target "OU=Admin,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "RDP-Restrict" -Target "OU=Servers,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "RPC-Restrict" -Target "OU=Servers,OU=NileCare,DC=nilecare,DC=local"
New-GPLink -Name "Update-Deferral" -Target "OU=Servers,OU=NileCare,DC=nilecare,DC=local"
```

### 5.1 AppLocker (edition: RESOLVED — design fix R11)
1. Edition trail (2026-11-18): backup VM = **Windows 10 Pro** (no enforcement); downloaded media = consumer Win11 25H2 ISO (no Enterprise) → **win-client = Windows 11 Education (Index 4), unactivated** (B0 §2), where these rules **enforce**. The mandatory early enforcement test (B0 §2 step 9) must have PASSED before this GPO's evidence is accepted. If anyone ever runs the GPO against the backup Win10 Pro VM, expect audit-mode events only — never present that as enforcement evidence.
2. GPO `AppLocker` → Windows Settings → Security Settings → Application Control Policies → AppLocker:
   - Create **Default Rules** for Executable, Windows Installer, Script, Packaged apps (Allow Everyone … built-ins).
   - Executable: Deny **BUILTIN\Users** everything under `C:\` except the default allows → effectively "allow only approved software".
   - Start in **Audit** mode (Config → Enforcement: Audit) for 3 days; verify event log `Applications and Services Logs → Microsoft → Windows → AppLocker`, then switch to **Enforce**.
   - Set service `AppIDSvc` trigger-start via GPO (Services setting).
3. Test evidence: copy `nmap.exe` to win-client → blocked at enforce (screenshot for scenario 9.3 support in Deliverable C).

## 6. dc02 — secondary DC + file server (§4.5, §4.6)
1. Build dc02 the same way as dc01: **D:\NileCare\vms\dc02 (D: HDD, cold set)**, 2 vCPU, 2048 MB, 24 GB thin, NIC → **VMnet17** (VLAN 70); ISO install (same verified EVAL ISO); rename dc02; IP `10.10.70.12/24`, DNS `10.10.70.11,10.10.70.12`; snapshot `dc02-pre-promo`; join domain.
2. Promote replica:
   ```powershell
   Install-WindowsFeature AD-DomainServices, DNS -IncludeManagementTools
   Install-ADDSDomainController -DomainName nilecare.local -InstallDns:$true `
     -SafeModeAdministratorPassword (Read-Host -AsSecureString) -Force
   ```
3. Replication checks: `repadmin /replsummary`, `repadmin /showrepl`, `dcdiag /s:dc02` → evidence `repadmin-replsummary.png`.
4. File services (dc02 is the file server — design decision log item 7):
   ```powershell
   Install-WindowsFeature FS-Resource-Manager, FS-SMB1 -IncludeManagementTools   # FSRM + SMB
   $shares = "ER","Nursing","Pharmacy","Billing","Radiology","IT","Backups"
   foreach ($s in $shares) {
     New-Item -Path "D:\Shares\$s" -ItemType Directory -Force
     New-SmbShare -Name $s -Path "D:\Shares\$s" -FullAccess "NILECARE\Domain Admins"
   }
   Grant-SmbShareAccess -Name ER -AccountName "NILECARE\GRP-Doctors-ER" -AccessRight Change -Force
   Grant-SmbShareAccess -Name Nursing -AccountName "NILECARE\GRP-Nurses" -AccessRight Change -Force
   Grant-SmbShareAccess -Name Pharmacy -AccountName "NILECARE\GRP-Pharmacists" -AccessRight Change -Force
   Grant-SmbShareAccess -Name Billing -AccountName "NILECARE\GRP-Billing" -AccessRight Change -Force
   Grant-SmbShareAccess -Name Radiology -AccountName "NILECARE\GRP-Radiology" -AccessRight Change -Force
   # NTFS: remove Authenticated Users inheritance; grant group Modify, Administrators Full
   icacls "D:\Shares\ER" /inheritance:r /grant:r "NILECARE\GRP-Doctors-ER:(OI)(CI)M" "NILECARE\Domain Admins:(OI)(CI)F" "SYSTEM:(OI)(CI)F"
   # repeat per share with its group
   ```
5. **FSRM — ransomware screening (§4.5, scenario 3 support)**:
   ```powershell
   $ext = @("*.locky","*.zepto","*.odin","*.cerber","*.crypt","*.wcry","*.onion","*.enc","*.encrypted","*.cryptolocker","*.ecc","*.zzz","*.paymen","*.locked","*.kraken","*.arena","*.radman","*.globe")
   $fs = New-FsrmFileGroup -Name "Ransomware-Extensions" -IncludePatterns $ext
   New-FsrmFileScreen -Path "D:\Shares" -IncludeGroup "Ransomware-Extensions" -Active:$true `
     -Description "Blocks known ransomware extensions" -NotificationLimit 1
   ```
   Verification: use the approved file-screening validation in Deliverable C and confirm that the event is recorded under `File Server Resource Manager → File Screening Management` → **fig-4.4-fsrm-file-screen.png**. The published test plan omits simulation commands.
6. Quotas: 5 GB soft quota per department share (`New-FsrmQuota -Path "D:\Shares\ER" -Size 5GB -SoftLimit`).

## 7. Windows Firewall + host baseline on DCs
- GPO Windows-Firewall applies automatically (gpupdate on both).
- Defender: keep real-time protection ON (part of the defence story); `Set-MpPreference -DisableRealtimeMonitoring $false`; scheduled quick scan Sunday 02:00 via GPO service or scheduled task.
- Verify the RPC restriction took effect: `Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Rpc\Internet` after gpupdate; then `Test-NetConnection dc01 -Port 135` OK, `Test-NetConnection dc01 -Port 49300` fails → evidence.

## 8. Patch management approach (§4.6, honest)
- **Chosen:** direct Windows Update with staged deferral GPO (Servers 3 d, Clinical 7 d) — a WSUS/SQL+WID installation does not fit dc02's 2 GB/24 GB budget (documented in Limitations 11.3). Weekly evidence: `Get-HotFix | Sort InstalledOn` output on dc01/dc02 into the evidence folder.
- WSUS remains documented as the production alternative.

## 9. win-client — domain join and validation (§4.1/§4.4 evidence)
1. Power on win-client (the **Windows 11 Education, unactivated** VM built in B0 §2 — 4 GB, NIC1 on VMnet10, hostname `WIN-CLIENT01`, snapshot `baseline-before-lab`, early AppLocker test must pass before this step). `ipconfig /all` → must get **10.10.10.110** via **relay** (evidence of the relay working — screenshot; the DHCP reservation ClientId must match the MAC recorded in B0 §2 step 8 — `<VERIFY-MAC-AFTER-NIC-SWAP>`).
2. Join domain (System Properties → Domain `nilecare.local` with adm.it1); move computer object to `OU=Clinical/Computers`; `gpupdate /force`; restart.
3. Log on as `omar.hassan` → H: mapped, lock screen at 5 min, USB write blocked, `gpresult /r` → **fig-4.3-gpresult.png**.
4. **AppLocker enforcement evidence (Education edition — enforced):** copy a non-approved `.exe` to the desktop → launch → **blocked** dialog; capture `applocker-block.png` (event log: `Applications and Services Logs → Microsoft → Windows → AppLocker → EXE and DLL`). Record the passing early local test (B0 §2 step 9) before testing this domain GPO; this step proves the **domain GPO** enforces it. Scenario 3/9 evidence uses this VM.
5. Wazuh agent install comes in B7 §4 (via Ansible).

## 10. Optional items — honest scope
- **AD CS / Windows Admin Center (§4.7): not implemented** — the 180-day eval and RAM budget favour core services; documented as future work (would give LDAPS-friendly PKI and browser-based management).

## Evidence register (chapter 4)

| File | Content | Template figure |
|---|---|---|
| fig-4.1-aduc-ou-structure.png | ADUC tree | Figure 4.1 |
| fig-4.2-dhcp-scopes.png | both scopes + relay | Figure 4.2 |
| fig-4.3-gpresult.png | `gpresult /r` on win-client | Figure 4.3 |
| fig-4.4-fsrm-file-screen.png | FSRM file-screen event | Figure 4.4 |
| dcdiag-dc01.txt / repadmin-replsummary.png | replication health | §4.6 |

**Power policy:** dc02 and win-client off after this chapter; snapshot `dc02-baseline`, `win-client-baseline` (single baseline snapshot each, per A §5 policy).
