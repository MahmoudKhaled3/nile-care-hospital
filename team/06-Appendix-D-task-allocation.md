# Appendix D - Team Contributions and Acceptance Ownership

This appendix replaces the original three-member placeholder allocation. The working names below must be replaced with official university spellings and student IDs before submission.

| Member | Contribution | Primary files/sections | Acceptance evidence | Integration dependency |
|---|---|---|---|---|
| Mahmoud | Lead integration; VMware networks; pfSense; Windows Server; dc01/dc02; GPO/AppLocker; Windows backup; final lab laptop and technical sign-off | B0; B4; pfSense side of B6; B8 §§2-5; C scenarios 1, 2, 7 and final orchestration | VM settings; pfSense version/interfaces; dcdiag/DNS/DHCP/GPO outputs; gate log; final technical checklist | Requires all accepted handoffs before final release |
| Youssef Al-Sayed | Four Linux VMs (emr01, pacs01, portal01, lin-client), Ansible, MariaDB and Linux backup | B5, B7 §5, B8 §§1 and 3, Linux portions of C | Playbook syntax/run, service versions, replication and backup outputs | Base siem01 from Shaker for Ansible, Gate 2 for AD join, Gate 4 for replication |
| Iprahim Said | Azure pre-flight; VNet/NSG; cloud-dr; strongSwan cloud side; Blob immutability; Azure Monitor/cost controls | B6; cloud portions of B8 and C scenario 7/8 | Azure region/SKU/quota; resource outputs; tunnel/cloud logs; immutability and monitor evidence | Requires approved IP plan and Youssef's replication prerequisites |
| Mahmed Salh | Synthea and public anonymised DICOM provenance; Chapters 1-2; references; compliance and policy review | E Chapter 1; Chapter 2 based on A and the template; D review; data manifest; references | Source/licence/hash manifest; reviewed legal references; synthetic-data declaration; no invented measurements | Requires final architecture facts from Mahmoud and service scope from Youssef |
| Mahmed Shaker | siem01 (4 GB), Wazuh manager and agent coordination, diagrams, evidence and report assembly | B5 §1 and §7.4, B7 §4, Wazuh config in B8 §3, C, Appendices C/D | Base VM handoff, dashboard, agent and VIP alert evidence, report QA and assembled source | Base siem01 ready before Gate 3. Integrate Wazuh in Gate 5 with the host owners, then obtain Mahmoud's sign-off. |

## Required contribution report

Each member submits a report after completing their package. It must answer:

```text
Member:
Package:
Planned items completed:
Items not completed:
Exact files and paths:
Commands actually run:
Evidence filenames:
Measured results (or <PENDING>):
Errors encountered and fixes:
Deviation from A-design-package.md:
Dependency handed to:
Reviewer and review date:
```

## Acceptance rule

Mahmoud checks technical consistency. Mahmed Shaker checks that evidence files exist and are mapped. Mahmed Salh checks sources and data claims. A package is accepted only when its owner, reviewer, and status are recorded. `STAGED` and `PARTIAL` are not gate passes.
