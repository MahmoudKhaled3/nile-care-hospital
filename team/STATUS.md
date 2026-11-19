# Nile Care project status

Last reviewed: 2026-11-19

This is the supervisor-facing summary. Package states are `PENDING`, `IN PROGRESS`, `READY FOR REVIEW`, `ACCEPTED`, `PARTIAL`, or `BLOCKED`; evidence maturity is `NOT RUN`, `STAGED`, `REPRODUCED`, or `MEASURED`. `ACCEPTED` means the owner's Pull Request was independently reviewed and merged into `main`. Checklist items use `DONE`, `PARTIAL`, `BLOCKED`, or `PENDING`. No state is a gate pass without the required evidence.

| Owner | Package | Package state | Evidence maturity | Tracking issue | Latest PR | Current result / next action |
|---|---|---|---|---|---|---|
| Mahmoud | fw01, VMware networks, Windows, final integration | `PARTIAL` | `STAGED` | [Issue #1](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/1) | `<PENDING>` | Gate 1: VMnet10 and VMnet16 have screenshots from the earlier host; VMnet12/13/14/15/17 remain unverified. Recheck all networks read-only on the final laptop before sign-off. |
| Youssef Al-Sayed | emr01, pacs01, portal01, lin-client, Ansible | `PENDING` | `NOT RUN` | [Issue #2](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/2) | `<PENDING>` | Submit staging inventory, service versions, playbook status, and the Linux handoff report. |
| Iprahim Said | cloud-dr, Azure networking, VPN, Blob, Monitor | `PENDING` | `NOT RUN` | [Issue #3](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/3) | `<PENDING>` | Submit Azure region/SKU/quota, cost estimate, VNet/NSG, tunnel, immutability, and deallocation evidence. |
| Mahmed Salh | Synthea/DICOM provenance, Chapters 1-2, references, compliance | `PENDING` | `NOT RUN` | [Issue #4](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/4) | `<PENDING>` | Submit source/licence/hash manifest and reviewed references. |
| Mahmed Shaker | siem01, Wazuh, diagrams, evidence, report QA | `PENDING` | `NOT RUN` | [Issue #5](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/5) | `<PENDING>` | Submit base siem01 handoff, evidence register, and report QA notes. |

## Gate state

| Gate | State | Evidence / next action |
|---|---|---|
| Gate 1 - VMware networks and fw01 | `PARTIAL` | VMnet10 and VMnet16 screenshots exist outside this repository; verify VMnet12/13/14/15/17 read-only before sign-off. |
| Gate 2 - Windows and AD | `<PENDING>` | Requires accepted Mahmoud Windows handoff and actual dcdiag/DNS/DHCP evidence. |
| Gate 3 - Linux base and Ansible | `<PENDING>` | Requires Youssef handoff and Shaker's base siem01 handoff. |
| Gate 4 - Azure and IPsec | `<PENDING>` | Requires Iprahim cloud evidence and Youssef replication prerequisites. |
| Gate 5 - Wazuh and integrated monitoring | `<PENDING>` | Requires Shaker's Wazuh evidence with Mahmoud's Suricata/pfSense inputs. |
| Gate 6 - Backup, restore, and release | `<PENDING>` | Requires measured restore evidence, report QA, and Mahmoud final sign-off. |

## Update rule

Every work PR updates the relevant row with package state, evidence maturity, tracking issue, active PR, exact files, sanitised evidence summary, measured values or `<PENDING>`, errors/fixes, deviation from A, and next dependency. After the work PR is merged, Mahmoud opens a status-only PR to change the package state to `ACCEPTED`, link the merged PR, and close the tracking issue. The independent reviewer and review date are recorded in that status-only PR.
