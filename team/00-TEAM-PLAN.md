# Nile Care Hospital - Team Execution and Integration Plan

ابدأ بتوزيع الـbriefs الخمسة مع مرجع المشروع. محمود مسؤول عن تشغيل المعمل النهائي والدمج على لابه. كل عضو يسلّم مخرجاته وتقرير حالة من [النموذج الموحد](08-HANDOFF-TEMPLATE.md). استخدم [دليل الدمج](07-INTEGRATION-PLAYBOOK.md) لمعرفة إمتى كل جزء يدخل في التجميعة وإزاي يتراجع.

| Brief | First output to deliver |
|---|---|
| [Mahmoud](01-Mahmoud.md) | Final laptop capacity and Gate 1 status |
| [Youssef Al-Sayed](02-Youssef-Al-Sayed.md) | Staging inventory, first Ubuntu build notes and playbook status |
| [Iprahim Said](03-Iprahim-Said.md) | Read-only Azure pre-flight and cost estimate |
| [Mahmed Salh](04-Mahmed-Salh.md) | Data source manifest and outline for Chapters 1–2 |
| [Mahmed Shaker](05-Mahmed-Shaker.md) | siem01 base VM and staging status, plus the evidence register |

Status: working plan for a five-person team
Project owner and final integrator: Mahmoud
Architecture source of truth: `E:\NileCare\report\A-design-package.md`

## 1. Team Roster

Use these display names exactly in the working files. Official university names, student IDs, supervisor name, and phone numbers are still `<PENDING>` and must be filled from the university list.

| Member | Working name | Primary ownership | Practical load |
|---|---|---|---|
| 1 | Mahmoud | Lead integration, VMware/pfSense, Windows, final assembly | Heavy: final lab on Mahmoud's laptop |
| 2 | Youssef Al-Sayed | Four Linux VMs (emr01, pacs01, portal01, lin-client), Ansible and Linux backup | Heavy: independent laptop staging, then guided deployment on Mahmoud's laptop |
| 3 | Iprahim Said | Azure DR, strongSwan cloud side, Blob immutability, Azure Monitor | Cloud work; no access to Mahmoud's laptop required for most tasks |
| 4 | Mahmed Salh | Synthetic data, DICOM sources, Chapters 1-2, references, compliance review | Documentation and data preparation |
| 5 | Mahmed Shaker | siem01 (4 GB), Wazuh, evidence, diagrams, Chapter 9, report assembly and QA | Practical work: stage siem01 and Wazuh, then integrate on Mahmoud's laptop |

The spelling above follows the names supplied by the team. Do not append `ICT` to any name. Replace a working name only after the official university spelling is supplied.

VM ownership totals are Mahmoud 4 local, Youssef 4 local, Shaker 1 local, and Iprahim 1 on Azure. Salh owns data and documentation. All nine local VMs are integrated on Mahmoud's laptop. See the [demo schedule](07-INTEGRATION-PLAYBOOK.md#5-demo-power-schedule).

## 2. Rules Everyone Follows

1. `A-design-package.md` is the authority for IPs, VLANs, VMnets, ports, RAM, disks, operating systems, and technology choices.
2. No one changes an IP, port, VM size, operating system, firewall direction, or tool without recording the proposed change and getting Mahmoud's approval.
3. A measured result is written only after the command or test actually runs. Until then use `<PENDING>` or `[before/after]`.
4. Passwords, PSKs, private keys, storage keys, and vault passphrases never go into chat, Git, screenshots, or report text. Use `<SET-IN-VAULT>`.
5. Every screenshot, log, CSV, hash, or command output gets a unique filename and an entry in `E:\NileCare\evidence\` and the evidence register.
6. A VM built on a member's laptop is a staging artifact. VMnets do not travel with VMs. Mahmoud's laptop is the final lab and evidence host. Keep staging networks isolated and retain their configuration notes for rebuild.
7. A laptop result is labelled `STAGED`, `REPRODUCED`, `MEASURED`, or `BLOCKED`; `STAGED` is never reported as a passed gate.
8. The gates remain sequential: Gate 1 -> Gate 2 -> Gate 3 -> Gate 4 -> Gate 5 -> Gate 6.

## 3. Architecture and Resource Constraints

The on-host design has nine VMs and `17.75 GB` provisioned RAM with a `10.5 GB` concurrent cap. A was sized against a reference host measured at `15.9 GB` RAM. Mahmoud reports a ninth-generation Intel Core i7 and `16 GB RAM` on the final laptop. Exact CPU model, usable memory, virtualization support and free space are `<PENDING>`. Use the smaller of the approved `10.5 GB` cap and the amount the laptop can actually spare. Validate the necessary phase on the laptop before scheduling it. The Azure `cloud-dr` VM is outside the on-host cap.

Youssef's laptop is reported as `8 GB RAM`; CPU, free disk, and hardware virtualization status are `<PENDING>`. It must not run the full stack. Stage one Ubuntu VM or configuration at a time, use WSL only for syntax checks when useful, and label all results as staged until reproduced on Mahmoud's laptop.

## 4. Gate Ownership and Dependencies

| Gate | Primary | Support | Entry condition | Acceptance owner |
|---|---|---|---|---|
| Gate 1: VMnets and fw01 | Mahmoud | Mahmed Shaker for evidence | VMnet screenshots/status and fw01 installation path available | Mahmoud; Shaker records evidence |
| Gate 2: dc01, DNS, DHCP, OUs, GPO | Mahmoud | Mahmed Shaker | Gate 1 accepted. Verify early AppLocker separately before relying on AppLocker or its gated scenarios. | Mahmoud |
| Gate 3: Linux base and Ansible | Youssef | Shaker supplies base siem01, Mahmoud verifies reachability | Gate 2 healthy and Shaker's base VM handoff accepted | Mahmoud |
| Gate 4: VPN and MariaDB replication | Mahmoud for pfSense, Youssef for emr01, Iprahim for cloud-dr | Shaker for timeline | Gate 3 accepted and Azure pre-flight complete | Mahmoud |
| Gate 5: Wazuh agents and rules | Mahmed Shaker | Youssef supports Linux agents, Mahmoud supports Windows and pfSense, Iprahim supports cloud-dr | Gate 4 accepted and relevant paths available | Mahmoud |
| Gate 6: backup and restore | Mahmoud for Windows/pfSense; Youssef for Linux/DB; Iprahim for Blob | Shaker for Wazuh config backup and evidence | Gate 5 agents and alert paths available | Mahmoud |

No member may declare a gate passed alone. The owner provides the command output; Mahmoud and Shaker check the evidence and the gate log.

## 5. Work-to-Deliverable Map

| Deliverable | Owner | Required handoff |
|---|---|---|
| A design package | Mahmoud, with all owners reviewing their sections | Approved IP/VM/port table; no unresolved design changes |
| B0 and B4 | Mahmoud | fw01 and Windows build records, settings screenshots, exact versions |
| B5 | Youssef for Linux and Ansible, Shaker for base siem01 and Wazuh agent configuration | Linux build notes, playbooks, inventory, baseline VM and syntax output |
| B6 | Iprahim plus Mahmoud/Youssef | Azure region/SKU/IP, VNet/NSG/Blob outputs, cloud strongSwan config with secrets removed |
| B7 | Shaker for Wazuh, Mahmoud for pfSense/Suricata, Youssef for Linux hardening | Configuration, agent list, rule outputs and limitations |
| B8 | Mahmoud for Windows/pfSense, Youssef for Linux and Ansible configs, Iprahim for Blob, Shaker for Wazuh config | Backup timers, replication, restore commands and measured timelines |
| C test plan | Mahmed Shaker | Empty result tables first; measured values only after execution |
| D policies | Mahmed Salh with Mahmoud review | Policy-to-control-to-evidence mapping; official names/contacts pending |
| E report text | Mahmed Shaker integrates; Salh owns Ch. 1-2 and references | Final text with no guidance boxes and no invented results |

## 6. Handoff Contract

Every member submits one `HANDOFF-<name>.md` using [the handoff template](08-HANDOFF-TEMPLATE.md). Include one row per deliverable and mark it done, partial, blocked, staged, or not run. Give the receiving owner the exact version, evidence and remaining dependency for each row.

The owner sends the handoff to Mahmoud and Mahmed Shaker. Shaker checks that every evidence filename exists; Mahmoud checks technical consistency. A handoff is accepted only when both checks are recorded.

## 7. Final Project Assembly

Mahmoud owns the final merge on his laptop. The assembly order is below. See [the integration playbook](07-INTEGRATION-PLAYBOOK.md) for detailed acceptance and recovery.

1. Freeze A and the approved change register.
2. Import each owner's handoff and copy only reviewed scripts/configuration into `E:\NileCare\repo`.
3. Recreate VMnets on the final host; never assume a member laptop's VMnet IDs or MACs are portable.
4. Reproduce the gates in order and write `evidence/gates.log` under the final project's root.
5. Shaker registers every evidence file and maps it to the required figure/table.
6. Salh verifies data sources, legal references, synthetic-data wording, and the no-real-patient-data declaration.
7. Shaker merges chapters into the official template, removes guidance boxes, updates table/figure numbering and the TOC, and runs the final consistency checklist.
8. Mahmoud performs the technical release review: IPs, ports, VM sizes, versions, firewall direction, backups, and gate status.
9. The team signs the release checklist. Any `<PENDING>`, missing evidence, or blocked test remains visible in the report's limitations.

## 8. Release Checklist

- [ ] Official team spellings, student IDs, supervisor, and contact fields filled.
- [ ] Five-person allocation appears in Appendix D and the working plan.
- [ ] No remaining statement says the project has three team members.
- [ ] All A/B/C/D/E references use the same IP, VLAN, VMnet, port, RAM, and OS values.
- [ ] Gate 1 is evidenced on the final laptop. Earlier VMnet10/16 screenshots apply to the earlier host only, where VMnet12/13/14/15/17 were still unverified.
- [ ] Every measured field is real or marked `<PENDING>`.
- [ ] No secrets appear in files or screenshots.
- [ ] Evidence register points to files that actually exist.
- [ ] Scenario phases stay at or below `10.5 GB`.
- [ ] Final report includes the five roles, limitations, references, figures, tables, and appendices.

## 9. Current Open Administrative Items

- Mahmoud's official full name and student ID: `<PENDING>`.
- Youssef, Iprahim, Mahmed Salh, and Mahmed Shaker official spellings and IDs: `<PENDING>`.
- Supervisor name and contact: `<PENDING>`.
- Defense date: `<PENDING>`; work is sequenced by dependencies rather than an invented calendar.
- Youssef laptop CPU, free disk, and virtualization support: `<PENDING>`.
- Mahmoud laptop usable RAM, exact CPU model, free disk, virtualization status and effective lab path: `<PENDING>`.

## 10. Template-to-Design Corrections

Mahmed Shaker uses this list while assembling the official template:

| Template wording or old assumption | Final project position |
|---|---|
| Windows Server 2022 | Windows Server 2025 Evaluation, build recorded at installation |
| OPNsense / Rocky / Zabbix | pfSense CE / Ubuntu Server 24.04 LTS / Wazuh; discarded names are not reintroduced |
| Orthanc on emr01 | Orthanc is on pacs01; emr01 is OpenEMR + MariaDB |
| SELinux | Ubuntu AppArmor + UFW + Fail2ban |
| OpenVAS | Nessus Essentials, with before/after counts left pending until measured |
| WSUS | Staged update-deferral GPOs in this constrained lab |
| AWS or unspecified cloud | Azure for Students, subject to the recorded region/SKU pre-flight |
| Missing VLAN 70/90 | VLAN 70 Servers and VLAN 90 OpenVPN tunnel are included in A |
| All clinical data from Synthea | Synthea supplies EMR records; DICOM comes from public anonymised datasets |
| DFS-R | Not implemented; remain in Limitations/Future Work |

Scenario 8 dependency review: C §8 currently lists `dc02` as on, but the listed EMR dump restore steps do not use it directly. Proposed set for review is `fw01 + emr01 + siem01` (7 GB), subject to verifying backup access and any DNS/auth dependency. Keep the runbook as written until this is reconciled. `dc02` remains necessary for Scenario 1 Windows shares, Scenario 3 FSRM, Windows System State/file backups, and its own scan batch.
