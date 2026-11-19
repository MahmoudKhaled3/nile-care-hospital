# Deliverable E — Report Text (Academic English for the report body)
**Secure Hybrid Enterprise Infrastructure — Nile Care Hospital**
Version 1.0 — 2026-11-19. Text is ready to paste into the template chapters; **fill-ins appear as [SQUARE BRACKETS]** and every measured number stays a placeholder until the Chapter 9 scenarios run. Abstract is written last per the template guidance.

---

# Abstract

Hospitals face significant operational risk from ransomware because downtime can interrupt clinical care and patient data requires protection. This project proposes a secure hybrid infrastructure for a modelled 250-bed hospital, using Windows Server 2025 domain services, Ubuntu 24.04 LTS for OpenEMR and Orthanc, pfSense network segmentation, and an Azure disaster recovery site. The design combines Suricata, Wazuh, Ansible hardening and multifactor authentication with MariaDB replication over IPsec and nightly encrypted backups to immutable storage. The target recovery point for EMR replication is 15 minutes. Nine planned scenarios cover identity, segmentation, monitoring, recovery and vulnerability management. Measured outcome fields are pending: EMR failover [minutes], data loss [records and elapsed time], backup restoration [minutes], Nessus findings [before/after], and Lynis indices [before/after]. The project includes security policies aligned with ISO/IEC 27001:2022 and Egypt’s Personal Data Protection Law.

**Keywords:** Active Directory, OpenEMR, pfSense, Wazuh, Suricata, Ansible, Disaster Recovery, Ransomware, Hospital IT.


---

# 1. Introduction

## 1.1 Background

Modern hospitals deliver care through interconnected information systems: electronic medical records at the point of care, picture archiving for radiology, laboratory interfaces, and pharmacy modules — all anchored on a directory service that decides who may see what. A failure of these systems is not merely an IT inconvenience; it is a clinical safety issue. Emergency departments divert ambulances, clinicians fall back to paper, and historical records become unreachable precisely when they are most needed.

The threat landscape matches this dependency. Ransomware groups deliberately target healthcare organisations because clinical downtime creates pressure to pay, and because patient data carries a durable black-market value; the Verizon Data Breach Investigations Report consistently ranks ransomware and credential abuse among the dominant attack patterns against organisations, and the IBM Cost of a Data Breach Report provides a benchmark for breach costs [1][2]. ENISA’s analysis documents ransomware and disruption across the European health sector [3]. At the same time, privacy regulation has hardened: Egypt's Personal Data Protection Law No. 151 of 2020 (PDPL) imposes licensing, security-of-processing, breach-notification (generally within 72 hours under Article 7) and cross-border transfer constraints on controllers of personal data — and health data is explicitly sensitive.

A hospital therefore needs an infrastructure that assumes attack: segmented so that a compromised workstation cannot reach the servers, monitored so that an intrusion is visible, backed up in a form ransomware cannot destroy, and rehearsed so that recovery is measured rather than hoped for. This project designs such an infrastructure for laboratory implementation and testing.

## 1.2 Problem Statement

Nile Care Hospital (250 beds, ≈600 staff, New Cairo) is modelled with a typical legacy starting point:

- a **flat network** in which clinical workstations, administrative clients, medical devices and servers share one subnet — any compromised endpoint reaches every server;
- **no disaster recovery**: the EMR and file services exist on single servers with untested copies;
- **unmanaged accounts**: shared administrative credentials, no role-based access, no audit trail of who opened which patient record;
- **no monitoring or IDS**: intrusions and brute-force attempts are invisible;
- **regulatory exposure**: sensitive health data processed without the controls the PDPL expects, and backups that would not survive a ransomware event that deletes the backup share itself.

The problem this project addresses is to transform this starting point into a defensible hybrid infrastructure — on-premises identity and services, a Linux clinical stack, and a cloud disaster-recovery site — and to **prove** the defences with measurable tests rather than assert them.

## 1.3 Objectives

1. Deploy centralised **identity, DNS, DHCP and file services** on **Windows Server 2025** (two domain controllers, GPO-based baselines, FSRM ransomware screening).
2. Deploy a **Linux-based EMR and PACS** — OpenEMR and Orthanc on Ubuntu Server 24.04 LTS — integrated with Active Directory and isolated in dedicated server (VLAN 70) and medical (VLAN 30) segments, behind a DMZ patient portal.
3. Build a **cloud disaster-recovery site** (Azure free/student tier) connected by a site-to-site IPsec VPN, with MariaDB replication delivering a 15-minute EMR RPO and **immutable** object storage for backups.
4. Apply **layered security**: server-segment firewalls, Suricata IDS, Wazuh SIEM with file-integrity monitoring, CIS-aligned hardening on Windows (GPO) and Linux (UFW, AppArmor, Fail2ban), MFA for remote access, and AppLocker application control.
5. **Validate the design** against measurable RTO/RPO targets using nine documented scenarios, including file-screening and recovery validation and a full EMR failover to the DR site.
6. **Govern the system**: eight security policies mapped to ISO/IEC 27001:2022 Annex A controls and the PDPL, so the technical design has an ownership and enforcement layer.

## 1.4 Scope and Limitations

**In scope:** the complete design, build, hardening, and testing of a nine-virtual-machine hospital environment (firewall, two Windows Server 2025 domain controllers, three Ubuntu servers for EMR/PACS/portal, a Wazuh SIEM, two test clients) plus an Azure DR virtual machine and immutable storage; a four-playbook Ansible automation layer; eight security policies; and nine test scenarios with pass/fail criteria.

**Simulated:** medical modalities are simulated with DICOM tooling (dcmtk) rather than real imaging devices; wireless and switching are represented by hypervisor virtual networks; all patients are synthetic (Synthea) and all imaging originates from public anonymised datasets. **Constraints:** the design fits a measured 15.9 GB host under an explicit power-state policy (concurrency cap 10.5 GB), uses evaluation/licensing-light software (Windows Server 2025 Evaluation, unactivated Windows 11 Education client, Azure student credit), and stores VMs on a consumer SSD. The complete limitation register appears in §11.3; each limitation states its compensating control.

## 1.5 Methodology

The project follows a plan → design → build → secure → test → document cycle. The design was fixed first in a versioned design package (VLAN/IP plan, firewall matrix with a companion port table, VM sizing fitted to the measured host, risk register) so that every later artifact remained consistent. The planned builds follow step-by-step runbooks with **stage gates**: a gate must pass (dcdiag healthy, Ansible syntax-checked, tunnel up, agents reporting, restore verified) before the next subsystem is built. Configuration is codified — four Ansible playbooks, GPO baselines, systemd timers, and a versioned repository with a secrets policy — so the environment is reproducible rather than hand-made. Security controls are scheduled alongside each subsystem build. Finally, nine test scenarios with explicit pass criteria measure the design's behaviour, including timed disaster-recovery exercises using a timestamped incident log, and a vulnerability comparison before and after hardening. Five team members are assigned lead integration/Windows, Linux/automation, Azure DR, data/compliance, and SIEM/evidence/release workstreams respectively, with shared evidence standards. Appendix D records the detailed task allocation.

## 1.6 Legal and Regulatory Framework

**Table 1.1: Frameworks referenced**

| Framework | Relevance to this project |
|---|---|
| Egypt Personal Data Protection Law No. 151 of 2020 (PDPL) | Consent and sensitive-data (health) rules; controller registration and DPO; 72-hour breach notification to the PDPC (Art. 7); cross-border transfer permit (Art. 14) — addressed by the Data Protection Policy (§10.4) and discussed for the DR site in §6.6 |
| HIPAA Security Rule | Administrative, physical and technical safeguards for health data (access control, audit controls, transmission security) — alignment reference for the clinical policies |
| ISO/IEC 27001:2022 (Annex A) | Control catalogue used to structure the eight security policies (Ch. 10) and the defence-in-depth architecture (Ch. 7) |
| CIS Benchmarks (Windows Server 2025, Ubuntu 24.04) | Technical hardening baselines implemented via GPO and the Ansible hardening playbook |

## 1.7 Report Structure

Chapter 2 analyses requirements and risk. Chapter 3 presents the architecture: VLAN and IP plan, VMware network model, VM sizing fitted to the measured host, the inter-VLAN firewall matrix with its port table, technology justifications, and the risk register. Chapter 4 documents the Windows Server 2025 implementation (domain, DNS/DHCP with a relay, OUs and GPOs, file services with FSRM). Chapter 5 documents the Linux stack (OpenEMR, Orthanc, the DMZ portal, SSSD domain integration, Ansible automation). Chapter 6 documents the Azure DR site, the IPsec tunnel, immutable storage and the data-residency analysis. Chapter 7 consolidates the security implementation. Chapter 8 defines backup and disaster-recovery procedures, including the timed failover runbook. Chapter 9 reports the nine test scenarios with measured results. Chapter 10 states the security policies. Chapter 11 concludes.

---

# 11. Conclusion and Future Work

## 11.1 Summary of Achievements

The design and runbooks address the objectives of §1.3. Implementation and validation status remain pending.

1. **Windows services.** The design specifies two domain controllers, AD-integrated DNS, a DHCP relay and Group Policy. Deployment and client evidence: [pending].
2. **Linux clinical stack.** The design specifies OpenEMR, Orthanc, directory integration and a DMZ portal. Service and sign-in evidence: [pending].
3. **Cloud disaster recovery.** The design specifies MariaDB replication over IPsec and encrypted, immutable backups. Failover time: [minutes]. Data loss: [records and elapsed time].
4. **Layered security.** Segmentation, IDS, SIEM, multifactor authentication and file screening are specified. AppLocker, FSRM, alerts and restore results: [pending].
5. **Validation.** Nine scenarios define the acceptance criteria. Nessus findings [before/after] and Lynis indices [before/after] await testing.
6. **Governance.** Chapter 10 records the proposed policies and maps them to the relevant controls.

## 11.2 Challenges and Lessons Learned

- **Sizing is a design input, not an afterthought.** The measured 15.9 GB host forced explicit trade-offs: a 4 GB SIEM with a tuned indexer heap, a concurrency cap of 10.5 GB with per-phase power states, and moving the PACS and client workloads to the slower disk. Lesson: fix the power-state matrix and the per-drive budget **before** the first VM is built; retrofitting it causes rework.
- **Edition gates must run before builds.** The clinical test client's original image proved to be Windows 10 **Pro**, which cannot enforce AppLocker. The consumer Windows 11 25H2 media then turned out to ship no Enterprise edition at all. The final position — Windows 11 **Education** running unactivated, gated by a mandatory early enforcement test — retains a path to collect AppLocker enforcement evidence. Lesson: verify licensing-relevant facts (edition, activation, expiry) as part of the design, with a documented fallback.
- **Distribution channels move.** The classic pfSense CE ISO mirror had been retired and the official download now flows through Netgate's store installer, which fetches the firewall over the internet at install time. Lesson: verify download paths and checksums at design time (the installer's SHA256 matched Netgate's published checksums file), and record the resulting version rather than assuming it.
- **Hypervisor limits are real.** VMware Workstation exposes only VMnet0–19 and 10 vNICs per VM; the original VMnet20–70 naming had to be remapped, and the eight-NIC firewall had to be shown to fit the documented limit.
- **Two recovery paths measure two different RPOs.** Replication targets a 15-minute RPO. The nightly dump targets a separate 24-hour RPO. Reporting them side by side (scenarios 7 and 8) is more honest — and more useful — than claiming a single number.
- **Automation discipline pays for itself.** Ansible playbooks and a secrets policy support reproducible builds. Syntax checks and a rebuild test remain pending.
- **Evidence integrity is a requirement, not a courtesy.** A "ransomware" demonstration that encrypts real data, or an AppLocker screenshot taken in audit mode, would invalidate the report. The simulation is designed to use a dedicated test share, and edition-dependent evidence is gated by an explicit enforcement test.

## 11.3 Limitations

1. **Environment fidelity.** VLANs are modelled with hypervisor host-only networks (Workstation cannot tag 802.1Q); the inter-VLAN firewall model is equivalent, but switch-level features (SPAN, DHCP snooping) are absent. The administrator's host holds a single adapter on the MGMT segment plus a static route — it is the evidence workstation. Scenario 6B uses it as the documented MGMT browser exception, and Scenario 9 uses it for scanning.
2. **Licensing and product lifecycles.** Windows Server 2025 runs under a 180-day evaluation; the win-client runs **Windows 11 Education unactivated** (watermark, no expiry) using the team’s consumer ISO — a mandatory early test demonstrates real enforcement before any AppLocker evidence is accepted. The team's original Windows 10 Pro VM (22H2, end-of-support since October 2025) is preserved untouched as a backup host and is outside the AppLocker test scope. Microsoft publishes no SHA256 for its evaluation/consumer ISOs; hashes are recorded locally and provenance is stated per file.
3. **Resource-driven consolidation.** dc02 combines the secondary-DC and file-server roles to stay within the host budget — in production these roles are separated (a file-server compromise must not expose AD DS on the same host). Ansible shares siem01 with Wazuh; the Wazuh node runs at 4 GB with a tuned indexer heap (documented as undersized for production); Wazuh indexes are not backed up. The 10.5 GB concurrency cap requires phased FSRM, VIP audit and vulnerability validation. FSRM enforcement and SIEM collection can occur in separate windows, correlated by the original event timestamps. The VIP alert phase uses a browser on the MGMT host, with the access-control phase on the Windows guest. Vulnerability counts compare matched batches of powered-on systems. These exercises do not measure simultaneous full-estate performance.
4. **Service substitutions.** WSUS is replaced by staged update-deferral GPOs (a WSUS/WID installation does not fit dc02's budget); Nessus Essentials (≤16 targets) replaces OpenVAS because the host cannot afford a GVM virtual machine — the tool change is stated wherever results appear; the DR VPN uses strongSwan on the cloud VM because Azure VPN Gateway is paid, and the on-premises endpoint's dynamic home IP is a documented operational caveat.
5. **Testing scope.** Medical modalities, Wi-Fi and switching are simulated; the ransomware simulation exercises FSRM screening, alerting and immutable restore (a harmless create/rename pattern), not real malware behaviour; Suricata's signature coverage for guest-side scans varies, and the scenario records what actually fired; OpenVPN inbound testing is limited by home-router NAT ("tested from LAN only" where applicable).
6. **Compliance posture.** The lab processes only synthetic data, so PDPL duties (registration, DPO, breach notification) are addressed in the policy and have not been exercised; production adoption requires PDPC registration, a DPO appointment, and the §6.6 residency decision.
7. **Remote management.** WinRM operates over HTTP (5985) inside the segmented lab; production requires TLS (5986). pfSense configuration backup is a monthly manual export (CE lacks built-in scheduled export).

## 11.4 Future Work

- **Endpoint detection and response** on all clients and servers (Defender for Endpoint or Wazuh XDR integration) with automated containment playbooks.
- **Automated DR failover**: health checks driving the DNS cutover and replica promotion (keepalived-style probes) instead of the manual timed runbook.
- **Network access control (802.1X)** with RADIUS so unmanaged devices cannot reach any internal VLAN regardless of port.
- **HL7/FHIR integration** and a Synthea→OpenEMR FHIR import to replace the manual patient-registration simplification.
- **In-country DR provider** with PDPC registration and a formal DPO process, replacing the student-credit cloud once patient data becomes real.
- **Role separation on dedicated hardware**: a dedicated file server (fs01) and a WSUS instance once a ≥32 GB host is available; DFS-R for departmental shares.
- **Enterprise licensing and MDM**: Windows 11 Enterprise with Intune-managed AppLocker baselines across all workstations, plus Entra-ID/RADIUS MFA replacing lab-scale TOTP.
- **SIEM maturation**: a clustered Wazuh deployment, longer index retention, and automated threat-intel feeds for Suricata/Wazuh.

---

## References used by this text (merge into the report's References section)

[1] IBM, *Cost of a Data Breach Report 2026*. https://www.ibm.com/reports/data-breach

[2] Verizon, *2026 Data Breach Investigations Report*. https://www.verizon.com/business/resources/reports/dbir/

[3] ENISA, *Health Threat Landscape* (2023). https://www.enisa.europa.eu/publications/health-threat-landscape

[4] Egypt, *Personal Data Protection Law No. 151 of 2020*, including Article 7 on breach notification and Articles 14–15 on cross-border transfers. English translation: Association of Corporate Counsel, https://www.acc.com/sites/default/files/program-materials/upload/Data%20Protection%20Law%20-%20Egypt%20-%20EN%20-%20MBH.PDF. The executive regulation citation is pending verification against the official text.

[5] ISO/IEC 27001:2022; CIS Benchmarks (Windows Server 2025, Ubuntu 24.04); HIPAA Security Rule (45 CFR §164).
[6] Vendor documentation cited per chapter: Netgate (pfSense install/checksums), OpenEMR, Orthanc, Wazuh, Ansible, Microsoft Evaluation Center.
