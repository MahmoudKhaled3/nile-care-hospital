# Nile Care project status

Last reviewed: 2026-11-19

This is the supervisor-facing summary. `ACCEPTED` means the owner's Pull Request was reviewed and merged into `main`. `STAGED`, `PARTIAL`, and `MEASURED` are useful progress states but are not gate passes by themselves.

| Owner | Package | State | Accepted source | Current result / next action |
|---|---|---|---|---|
| Mahmoud | fw01, VMware networks, Windows, final integration | `PARTIAL` | [Issue #1](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/1) | Gate 1: VMnet10 and VMnet16 verified by screenshot; VMnet12/13/14/15/17 remain unverified. Continue read-only verification before sign-off. |
| Youssef Al-Sayed | emr01, pacs01, portal01, lin-client, Ansible | `PENDING` | [Issue #2](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/2) | Submit staging inventory, service versions, playbook status, and the Linux handoff report. |
| Iprahim Said | cloud-dr, Azure networking, VPN, Blob, Monitor | `PENDING` | [Issue #3](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/3) | Submit Azure region/SKU/quota, cost estimate, VNet/NSG, tunnel, immutability, and deallocation evidence. |
| Mahmed Salh | Synthea/DICOM provenance, Chapters 1-2, references, compliance | `PENDING` | [Issue #4](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/4) | Submit source/licence/hash manifest and reviewed references. |
| Mahmed Shaker | siem01, Wazuh, diagrams, evidence, report QA | `PENDING` | [Issue #5](https://github.com/MahmoudKhaled3/nile-care-hospital/issues/5) | Submit base siem01 handoff, evidence register, and report QA notes. |

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

Every accepted package updates this table in the same merged PR with: owner, state, exact files, evidence filenames, measured values or `<PENDING>`, errors/fixes, deviation from A, dependency, reviewer, and review date.
