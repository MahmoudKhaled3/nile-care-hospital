# Deliverable D — Security Policies (report Chapter 10)
**Secure Hybrid Enterprise Infrastructure — Nile Care Hospital**
New Cairo Technological University | Server Administration Final Project
Version 1.0 — 2026-11-19. Report language: English. Each policy = 1–2 report pages (purpose → scope → statements → roles → enforcement).

---

## How to read this document

- **Framework mapping:** every policy cites **ISO/IEC 27001:2022 Annex A** control numbers (the report's §1.6 references ISO/IEC 27001; 2022 is the current edition) and the **Egypt Personal Data Protection Law No. 151 of 2020 (PDPL)**.
- **Verified legal anchors used throughout:** PDPC establishment and powers (**Art. 19**); sensitive data requiring express consent, including health and children's data (definition in Art. 1 and controls in Art. 12); **cross-border transfer prohibition except per PDPC/Executive-Regulation conditions (Arts. 14–15)**; controller/processor registration & DPO obligations, **72-hour breach notification to the PDPC** (**Law No. 151/2020, Art. 7**); penalties from fines to imprisonment for sensitive-data and unauthorised-transfer violations. Data-subject rights (access, rectification, objection) are cited per the Law's rights provisions; the **official Arabic text is authoritative — the final report cites it and the ACC/Andersen English translations as secondary sources.**
- **Lab note:** every policy also names the technical control that implements it in this project (GPO names, pfSense rules, Wazuh, FSRM, restic…) so the report can link policy → implementation → test evidence (Chapter 9). All lab data is synthetic (Declaration, report p. 2), so **no real personal data is processed in the lab** — the policies describe how the project would demonstrate the relevant controls.

---

## 10.1 Access Control Policy

**Purpose.** Ensure that access to Nile Care Hospital's information systems is limited to authorised individuals, is based on clinical and administrative roles, and is reviewable — protecting patient records (PDPL-sensitive data) and the hospital's clinical operations.

**Scope.** All information systems operated by the hospital: Active Directory domain `nilecare.local`, the EMR/HIS (OpenEMR), PACS (Orthanc), patient portal, file services, network infrastructure (fw01) and the DR environment. Applies to all 600 staff, contractors, and administrators.

**Policy statements**
1. Access is granted through **role-based security groups** (GRP-Doctors-ER, GRP-Nurses, GRP-Pharmacists, GRP-Billing, GRP-Radiology, GRP-IT-Admins); individuals never receive permissions directly. *(ISO A.5.15, A.5.16, A.8.3)*
2. **Least privilege:** each department group receives the minimum rights needed for its function; cross-department record access is denied by default and logged. *(ISO A.8.3; PDPL security-of-processing obligations)*
3. **Privileged access is separated** from daily accounts: IT staff use dedicated `adm.*` accounts (Domain Admins) and never use them for office work. Privileged rights are approved by the IT manager and reviewed quarterly. *(ISO A.8.2)*
4. **Network access is segmented:** user VLANs cannot reach servers except through the documented inter-VLAN policy; management traffic originates only from the MGMT segment. *(ISO A.8.22 — implemented by pfSense Tables 3.4/3.5)*
5. **Joining Linux servers to the domain** is permitted only for administrative hosts (emr01), with SSH limited to the IT-administrators group via SSSD. *(ISO A.5.18)*
6. Access rights are **recruited, changed and revoked** through the joiner/mover/leaver process; HR notifies IT the same working day a contract ends; accounts are disabled (never deleted) on exit and reviewed at 30 days. *(ISO A.5.16, A.5.18)*
7. A **break-glass account** (`svc-breakglass`) exists for DR events: disabled in normal operation, its use triggers a security alert (Wazuh rule 100130) and post-use review.
8. All access decisions to patient records are **logged and monitored** (OpenEMR audit log → Wazuh). *(ISO A.8.15, A.8.16; PDPL security-of-processing)*

**Roles.** IT manager approves privileged rights; department heads approve role membership; IT operations implement and review quarterly; the security analyst reviews access logs weekly.
**Enforcement.** Quarterly access reviews; violations follow the disciplinary process (10.3); emergency changes without approval are reviewed within 24 h.
**Implementation in this project:** OU/group design + user provisioning scripts (Ch. 4), SSSD access filter (Ch. 5), inter-VLAN matrix (Ch. 3), break-glass alerting (Ch. 7).

---

## 10.2 Password and Authentication Policy

**Purpose.** Guarantee that every identity is authenticated with strength proportional to the risk of the system it reaches, and that authentication secrets are never exposed.

**Scope.** All domain accounts, service accounts, local accounts, network devices (fw01), cloud access (Azure), and remote-access (OpenVPN) sessions.

**Policy statements**
1. **Password minimum:** length ≥ 12 characters, complexity on, history 12, maximum age 90 days; lockout after 5 failed attempts for 15 minutes. *(ISO A.8.5; implemented by GPO `Security-Baseline`, Ch. 4)*
2. **MFA is mandatory** for remote clinical access (OpenVPN with TOTP tokens in pfSense User Manager). Administrative access to infrastructure from outside the MGMT segment requires the VPN. *(ISO A.8.5)*
3. **Authentication secrets are stored only in the approved password vault** (KeePassXC); they never appear in scripts, tickets, screenshots, or committed files. Service-account passwords rotate at least annually and on staff departure. *(ISO A.5.17 — management of authentication information; see B-README "Secrets & repo hygiene")*
4. **Privileged authentication uses dedicated accounts** (statement 10.1.3); shared/generic accounts are prohibited except the break-glass account (10.1.7).
5. **Local administrator passwords** on managed endpoints are unique per machine and rotated (Windows LAPS backed up to AD, readable only by GRP-IT-Admins). *(ISO A.8.2)*
6. **Remote authentication is protected against brute force:** Fail2ban bans sources after 5 failed SSH attempts for 1 hour (Ch. 5/7), and pfSense blocks all other management paths (Ch. 3).
7. **Unactivated/evaluated clients must still authenticate to the domain** — the education-edition win-client complies with statement 1 through domain policy.
8. Authentication failures and anomalies are monitored (Wazuh rules; scenario 5 evidence).

**Roles.** IT operations own enforcement; users are accountable for their secrets; the security analyst investigates lockout/anomaly patterns weekly.
**Enforcement.** Non-compliant accounts are disabled after warning; vault breaches are treated as security incidents (10.5).
**PDPL note.** Credential compromise on systems holding patient data constitutes a processing-security violation → incident path 10.5 (incl. the 72-hour PDPC notification duty, PDPL Art. 7).

---

## 10.3 Acceptable Use and BYOD Policy

**Purpose.** Define acceptable use of hospital IT assets, and the conditions under which personal devices may touch hospital systems, protecting patient privacy and the clinical network.

**Scope.** All staff (clinical, administrative, IT), contractors, the Guest VLAN, and any personal device connecting to hospital Wi-Fi.

**Policy statements**
1. Hospital systems are provided **for professional use**; incidental personal use must not consume capacity or introduce risk. Prohibited: installing unapproved software (AppLocker allow-listing enforces this on managed clients — Ch. 4), bypassing security controls, connecting unauthorised devices to internal VLANs.
2. **Removable media are blocked on clinical and administrative clients** (GPO `Block-USB`); data leaves the hospital only through approved encrypted channels. *(ISO A.8.12-scope; ransomware defence, Ch. 4)*
3. **Patient data may not be stored on personal devices.** Clinical access from remote/personal devices happens only through the hospital VPN with MFA; the device must have disk encryption and a screen lock. *(ISO A.6.7, A.8.1; PDPL: controller remains responsible for data accessed by processors)*
4. **Personal devices (BYOD) receive internet-only access** on the Guest VLAN (VLAN 50): no routing to any internal segment (pfSense denies and logs — scenario 4 evidence), DHCP from dc01 via the firewall relay after Gate 1, DNS via the firewall resolver.
5. **Monitoring is disclosed:** authentication logs, endpoint protection events, network logs and the SIEM (Wazuh) process data about *users' system activity* for security purposes. Staff are informed at onboarding (this policy) that such monitoring exists and what it covers; content of personal communications is not inspected. *(PDPL transparency obligations for processing employee data)*
6. Security events (suspected malware, lost device, phishing) **must be reported within one working day** to IT (reporting channel = 10.8 contacts). *(ISO A.6.8-scope)*
7. Screen-lock inactivity (5 minutes) is mandatory on all clinical/administrative clients (GPO `Lock-Screen-5min`).

**Roles.** All staff acknowledge this policy at onboarding (signed); department heads ensure compliance; IT implements technical controls; HR applies disciplinary measures.
**Enforcement.** Violations range from access revocation to disciplinary action up to termination; malicious use of patient data is referred per the penalties regime of the PDPL.

---

## 10.4 Data Protection and Privacy Policy

**Purpose.** Protect the confidentiality and integrity of patient and staff personal data in line with **Egypt PDPL No. 151/2020**, HIPAA-aligned practices, and ISO/IEC 27001:2022.

**Scope.** All personal data processed by the hospital — including **health data, which is sensitive under PDPL Art. 7** and requires express consent — across EMR, PACS, file services, backups, and the DR environment.

**Policy statements**
1. **Registration:** the hospital, acting as controller, is registered/licensed with the **Personal Data Protection Centre (PDPC)** and maintains a **Data Protection Officer** contact, per Art. 4 and the Executive Regulations; processing inventories are kept current. *(ISO A.5.34)*
2. **Lawful basis:** patient processing relies on the Law's consent and permitted-case provisions; **sensitive-data processing requires express consent** (with the statutory exceptions for care delivery) and **children's data requires guardian consent** (children's data is sensitive per Art. 12). Consent records are retained.
3. **Minimum necessary:** clinical roles see only the data their function requires (10.1); bulk exports of patient data require DPO approval.
4. **Integrity & confidentiality safeguards:** encryption in transit (TLS everywhere — nginx configs, Ch. 5) and at rest for exports (GPG, Ch. 8); endpoint protection; segmentation of clinical data flows (Ch. 3). *(ISO A.8.24, A.8.22; PDPL security-of-processing)*
5. **Data-subject rights:** patients may request access, rectification, or to object to processing; requests are handled by the DPO within the statutory timeframe, with identity verification and an auditable trail.
6. **Deletion & retention:** personal data is kept no longer than the clinical/legal retention schedule (10.6); deletion is executed per ISO A.8.10 and verified.
7. **Cross-border transfer:** governed by §6.6 (dedicated discussion below) — **no transfer abroad without PDPC authorisation or a permitted exception**; violations carry criminal penalties. *(PDPL Arts. 14–15)*
8. **Breach handling:** any personal-data breach is handled as a P1 incident (10.5) with **notification to the PDPC within 72 hours** (immediately for national security matters) and affected-person communication under Article 7. *(PDPL Art. 7)*
9. **Processors** (cloud/hosting providers) are bound by written agreements and may not sub-process without authorisation; the hospital verifies their security (ISO A.5.19-scope; PDPL controller–processor duties).
10. **This lab uses only synthetic data** (Synthea + public DICOM) — no real personal data is processed; the runbooks specify controls for later demonstration.

**Roles.** DPO owns PDPL compliance; IT implements safeguards; clinical departments apply minimum-necessary access; the security analyst monitors.
**Enforcement.** PDPL penalties include administrative fines and criminal liability (imprisonment for unauthorised sensitive-data processing or unauthorised transfers); internal violations follow 10.3.

---

## 10.5 Incident Response Policy

**Purpose.** Detect, triage, contain, eradicate and learn from security incidents in a repeatable way, and meet the statutory notification duties.

**Scope.** All information-security events on any hospital system, on-premises or in the DR cloud.

**Policy statements**
1. **Detection** is continuous: Wazuh (agents on every server/client, custom rules), Suricata IDS (all segments), FSRM ransomware screening, syslog centralisation. *(ISO A.8.15, A.8.16)*
2. **Reported events** (user reports per 10.3.6, SIEM alerts, vendor advisories) are **triaged within 1 working hour** during clinic hours by the security analyst, using the severity scale P1–P4 defined in the IR plan.
3. **P1 incidents** (ransomware, confirmed breach of patient data, domain compromise) invoke the **DR runbook** (Ch. 8): isolate (VLAN disable/host block), identify scope (Wazuh FIM + FSRM events), eradicate, restore from the immutable copy, verify integrity.
4. **Notification:** personal-data breaches are reported to the **PDPC within 72 hours** (immediately for national security matters), and to affected persons as required by Article 7; the incident commander owns all notifications. *(PDPL; ISO A.5.26)*
5. **Evidence preservation:** logs, memory where feasible, and timeline files (`dr-timeline.log`) are collected before remediation alters state. *(ISO A.5.28)*
6. **Post-incident review** within 5 working days: root cause, detection gap, lessons (feeds report §9.4). *(ISO A.5.27)*
7. **Drills:** the ransomware scenario (Ch. 9, scenario 3) and the EMR failover (scenario 7) are rehearsed at least once per semester with the real runbooks.

**Roles.** See 10.8. The incident commander is the single decision authority; the security analyst leads triage/evidence; the systems engineer executes containment/recovery; the communications lead handles staff, management, and regulator notifications.
**Enforcement.** Failure to report or escalate is a policy violation; drills not executed are a management finding.

---

## 10.6 Backup and Retention Policy

**Purpose.** Guarantee recoverability of clinical and administrative data within the stated RTO/RPO objectives, and retain data per clinical and legal requirements without over-retention.

**Scope.** EMR database, Active Directory, file shares, PACS, configuration state (pfSense, Wazuh, Ansible), and the DR cloud.

**Policy statements**
1. **3-2-1 rule:** three copies, two media, one off-site and **immutable** — implemented per the backup matrix (Table 8.1): nightly dumps + continuous MariaDB replication; restic with client-side encryption to Azure Blob under a time-based immutability policy. *(ISO A.8.13, A.8.14, A.5.30)*
2. **Immutability:** backup objects are WORM-protected (365-day policy) so ransomware cannot delete them; deletion attempts are impossible by policy — verified in scenario 8.
3. **Encryption:** dumps are GPG-encrypted client-side; the private key never resides on the backup target or in any repository (B8 §1.2). *(ISO A.8.24; PDPL security-of-processing)*
4. **Testing:** backups are validated by restore, not by job success — at minimum the two DR scenarios per semester (Ch. 9 scenarios 7–8) plus a monthly spot-restore.
5. **Retention:** EMR dumps 90 days; system state 30 days; file shares 30 days; pfSense/Wazuh config 90 days; OVA cold archive 2 generations — **and patient data itself is retained per the clinical schedule**; expired backups are pruned automatically (restic `forget`), deleted patient data is removed from backups at the end of their retention window (ISO A.8.10; PDPL storage-limitation).
6. **Media handling:** backup keys and GPG private keys are vault-managed; archives leave-site only encrypted; disposal is by cryptographic erasure where media permits.
7. **Recovery objectives** follow Table 8.2 (EMR 1 h/15 min via replication; measured values recorded in Ch. 9).

**Roles.** IT operations own backup jobs; the DPO owns retention decisions for personal data; the incident commander may authorise emergency restores.
**Enforcement.** A missed backup window is a P2 event; failed verification blocks the weekly sign-off.

---

## 10.7 Patch and Vulnerability Management Policy

**Purpose.** Keep every system's vulnerability exposure measured and minimised through a repeating cycle of patch, verify and report.

**Scope.** Windows servers/clients, Ubuntu servers/clients, pfSense, network services, cloud instances.

**Policy statements**
1. **Inventory first:** every asset is known (Tables 3.3/2.2) before it can be patched; unknown devices on user VLANs are investigated (DHCP lease review).
2. **Patch cadence:** **security updates within 14 days** of release for servers, 30 days for clients — staged via update-deferral GPOs (Servers 3 d, Clinical 7 d, B4 §8); Ubuntu servers use `unattended-upgrades` plus a monthly maintenance window. pfSense follows release notes with a snapshot-then-patch procedure. *(ISO A.8.8-scope)*
3. **Vulnerability scanning:** authenticated scans at least **once per hardening cycle** (before/after) with the documented policy (Ch. 9 scenario 9); results are triaged — Critical ≤ 7 days, High ≤ 30 days, Medium ≤ 90 days, Low/informational reviewed.
4. **Exception handling:** a finding that cannot be patched gets a documented risk acceptance (owner + compensating control + expiry date) — e.g., the unactivated Education client is covered by compensating controls in 11.3.
5. **Baseline hardening:** every host applies the Ansible `hardening.yml` baseline and the Windows GPO baseline before joining production segments; Lynis indices are recorded before/after (Ch. 5/7).
6. **Change control:** patches to dc01/dc02/emr01 follow the snapshot-then-apply pattern (B4 §1, B7); failed patches roll back from snapshots.
7. **Reporting:** monthly patch/vulnerability status to management (evidence: `Get-HotFix`, Nessus exports, Lynis deltas).

**Roles.** IT operations execute patching; the security analyst owns scanning/triage; system owners accept exceptions.
**Enforcement.** Overdue Criticals are escalated to the incident commander; unscanned systems may not be connected to Servers/MGMT VLANs.

---

## 10.8 Incident Response Roles

**Table 10.1: Incident response team** *(student roles for the lab exercise. Official names and contact information are pending. Production DPO and regulator liaison follow the policy.)*

| Role | Responsibility | Contact |
|---|---|---|
| **Incident commander** | Overall decisions, severity assignment, simulated escalation, authorises DR invocation | Mahmoud — [official ID / phone pending] |
| **Security analyst** | Triage and investigation: SIEM triage, evidence preservation (`dr-timeline.log`), scope assessment, post-incident review | Mahmed Shaker — [official ID / phone pending] |
| **Systems engineer** | Containment and recovery: isolation, snapshot/restore execution, service rebuild per runbooks Ch. 8 | Mahmoud — [official ID / phone pending] |
| **Cloud/DR engineer** | Azure DR, strongSwan cloud endpoint, Blob immutability, cloud-side recovery support | Iprahim Said — [official ID / phone pending] |
| **Data and compliance lead** | Synthetic-data provenance, references, policy wording, drafts simulated staff and management updates for the exercise | Mahmed Salh — [official ID / phone pending] |
| **Evidence and release lead** | Evidence register, report assembly, test traceability, release QA and communications pack | Mahmed Shaker — [official ID / phone pending] |

**Escalation path:** detection (any staff/monitor) → security analyst (1 working hour) → incident commander for P1 → data and compliance lead for simulated communications. Out-of-hours: the on-call rotation covers the commander and engineer roles; the DPO is engaged immediately for any suspected personal-data breach.

---

## §6.6 (Ch. 6) — Data Residency and Cross-Border Transfer (PDPL discussion)

**Deliverable text for report §6.6:**

> **Data residency and cross-border transfer.** Egypt's PDPL prohibits transferring personal data abroad except under the conditions and controls set by the Executive Regulations and the Personal Data Protection Centre (Arts. 14–15), including a PDPC licence or permit under Art. 14; unauthorised transfer carries criminal penalties. Health data — the hospital's core dataset — is sensitive under Art. 1 and children's records are likewise classified as sensitive, so a 250-bed hospital cannot treat cloud location as a purely technical choice.
>
> **In this project the constraint is not triggered:** every record in the EMR, PACS and backups is synthetic (Synthea) or a public anonymised DICOM sample, so no personal data crosses any border. The design specifies the following controls for demonstration: client-side GPG encryption of every export (the cloud never receives plaintext keys), IPsec protection for the replication path and HTTPS for application access, least-data replication (only the EMR database, no imaging, no logs), and an immutable, region-pinned Azure storage account (UAE North where the subscription permits).
>
> **Production recommendation (ranked):** (1) a private in-country DR site — no transfer occurs and latency to Cairo is minimal; (2) an in-country cloud region operated by a PDPC-registered provider — the transfer is domestic, and the PDPL Art. 4 licensing duty still applies; (3) a cross-border region **only** with a PDPC permit, contractual safeguards with the provider, and encryption whose keys remain in Egypt — accepting that a hospital is accountable for its processor's failures. The design's controls (encryption, immutability, least-data) are transferable to whichever option the hospital selects; what changes is only the paperwork and the region. This analysis satisfies the PDPL discussion required for the hybrid design and is referenced by the Data Protection Policy (10.4, statements 7 and 9).

---

## References for this chapter (report §10 footer)

1. ISO/IEC 27001:2022, Annex A controls as cited per policy.
2. Egypt Personal Data Protection Law No. 151 of 2020 (PDPL) — official Arabic text (authoritative) and English translations: [ACC English translation](https://www.acc.com/sites/default/files/program-materials/upload/Data%20Protection%20Law%20-%20Egypt%20-%20EN%20-%20MBH.PDF) and [Andersen Egypt full-text translation](https://eg.andersen.com/wp-content/uploads/2025/06/Law-No.-151-OF-2020.pdf); see also [DLA Piper — Data protection laws in Egypt](https://www.dlapiperdataprotection.com/?t=law&c=EG) for the Executive Regulations summary.
3. PDPL Executive Regulations — official publication and article references pending verification.
4. HIPAA Security Rule (45 CFR §164) — administrative/physical/technical safeguards, referenced for clinical practice alignment.
5. CIS Benchmarks (Windows Server 2025, Ubuntu 24.04) — technical baselines referenced by policies 10.2/10.7.
6. Project implementation chapters: policies → controls → test evidence traceability (Ch. 3–9).
