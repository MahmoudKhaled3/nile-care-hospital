# GitHub delivery workflow

This repository is the supervisor-facing source of progress. The local VM environment and evidence folder remain outside GitHub unless a reviewed, sanitised summary is appropriate.

## One package at a time

| Owner | Branch prefix | Main package | Reviewer | First status target |
|---|---|---|---|---|
| Mahmoud | `work/mahmoud-` | fw01, dc01, dc02, win-client, integration | Mahmoud + Shaker evidence check | Gate 1 partial, then Gate 2 |
| Youssef Al-Sayed | `work/youssef-` | emr01, pacs01, portal01, lin-client, Ansible | Mahmoud technical check | Linux staging handoff |
| Iprahim Said | `work/iprahim-` | cloud-dr, VNet/NSG, strongSwan, Blob, Monitor | Mahmoud + Youssef dependency check | Azure pre-flight |
| Mahmed Salh | `work/salh-` | Synthea/DICOM manifest, Chapters 1-2, references, compliance | Shaker report check | Source manifest |
| Mahmed Shaker | `work/shaker-` | siem01, Wazuh, diagrams, evidence, report QA | Mahmoud technical sign-off | siem01 base handoff |

## Branch to merge

1. Start from the current `main` branch and create `work/<member>-<package>`.
2. Make focused commits. Keep the architecture values from `report/A-design-package.md` unchanged unless the PR records the proposed change and Mahmoud approves it.
3. Open a Pull Request into `main`. Use the PR template and update `team/STATUS.md` in the same branch.
4. Attach evidence by filename and relative path. Do not upload passwords, PSKs, private keys, vault files, VM disks, ISO media, raw screenshots, or real patient data.
5. The owner marks results `STAGED`, `REPRODUCED`, `MEASURED`, `PARTIAL`, or `BLOCKED`. Only Mahmoud's merge after review changes the supervisor-facing state to `ACCEPTED`.
6. After merge, add the next dependency and next action to `team/STATUS.md`. Close the matching issue or mark it `ACCEPTED`.

## What the supervisor sees

Open [`team/STATUS.md`](STATUS.md) for the current summary, then open the linked Pull Request for the exact files, evidence names, decisions, and remaining work. A missing measurement stays `<PENDING>`; a staged result is never presented as a passed gate.

## Access

The repository is private. Mahmoud adds team members with write access and the supervisor with read access by their GitHub usernames. Email addresses alone cannot grant repository access or be used for GitHub mentions. Until access is granted, a member sends Mahmoud their branch name or patch and does not share a password or token.
