# GitHub delivery workflow

This repository is the supervisor-facing source of progress. The local VM environment and raw evidence folder remain outside GitHub; a reviewed, sanitised evidence summary belongs in the PR and [`team/EVIDENCE-INDEX.md`](EVIDENCE-INDEX.md).

## One package at a time

| Owner | Branch prefix | Main package | Reviewer | First status target |
|---|---|---|---|---|
| Mahmoud | `work/mahmoud-` | fw01, dc01, dc02, win-client, integration | Youssef or Iprahim peer review + Shaker evidence check; Mahmoud final sign-off | Gate 1 partial, then Gate 2 |
| Youssef Al-Sayed | `work/youssef-` | emr01, pacs01, portal01, lin-client, Ansible | Mahmoud technical check | Linux staging handoff |
| Iprahim Said | `work/iprahim-` | cloud-dr, VNet/NSG, strongSwan, Blob, Monitor | Mahmoud + Youssef dependency check | Azure pre-flight |
| Mahmed Salh | `work/salh-` | Synthea/DICOM manifest, Chapters 1-2, references, compliance | Shaker report check | Source manifest |
| Mahmed Shaker | `work/shaker-` | siem01, Wazuh, diagrams, evidence, report QA | Mahmoud technical sign-off | siem01 base handoff |

## Branch to merge

1. Start from the current `main` branch and create `work/<member>-<package>`.
2. Make focused commits. Keep the architecture values from `report/A-design-package.md` unchanged unless the PR records the proposed change and Mahmoud approves it.
3. Open a Pull Request into `main`. Use the PR template, update `team/STATUS.md` in the same branch, and write the next dependency/action before requesting review.
4. Add a sanitised evidence summary and filenames. Do not upload passwords, PSKs, private keys, vault files, VM disks, ISO media, raw screenshots, or real patient data. See `team/EVIDENCE-INDEX.md`.
5. The owner marks checklist items `DONE`, `PARTIAL`, `BLOCKED`, or `PENDING`; package state `PENDING`, `IN PROGRESS`, `READY FOR REVIEW`, `PARTIAL`, or `BLOCKED`; and evidence maturity `NOT RUN`, `STAGED`, `REPRODUCED`, or `MEASURED`.
6. An independent teammate reviews the owner's work. Mahmoud gives final integration sign-off and merges. After merge, Mahmoud opens a small status-only PR to mark the package `ACCEPTED`, link the merged PR, record the next dependency, and close the tracking issue. Nobody edits `main` directly.

## What the supervisor sees

Open [`team/STATUS.md`](STATUS.md) for the current summary, then open the tracking issue and latest PR for the exact files, evidence summary, decisions, and remaining work. A missing measurement stays `<PENDING>`; a staged result is never presented as a passed gate.

## Access

The repository is public and contains only the reviewed, sanitised project package, so the supervisor can view it without access setup. Mahmoud adds team members with write access by GitHub username. Email addresses alone cannot grant repository access or be used for GitHub mentions. Until write access is granted, a member sends Mahmoud their branch name or patch and does not share a password or token.

## Main-branch enforcement check

Before the first team PR, Mahmoud must open **Settings → Branches** and confirm that `main` requires a pull request, at least one approval, conversation resolution, and no force-pushes. The current connected GitHub integration could not verify this setting remotely, so treat the rule as unconfirmed until Mahmoud sees it in the repository UI.
