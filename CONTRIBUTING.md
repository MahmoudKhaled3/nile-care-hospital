# Contributing to Nile Care

## Before starting

1. Read `report/A-design-package.md` and the member brief under `team/`.
2. Confirm the dependency and acceptance evidence in `team/09-GITHUB-WORKFLOW.md`.
3. Never commit passwords, PSKs, private keys, vault files, VM disks, ISO media, raw evidence, or real patient data.

## Branch and commit

Create one branch per work package:

```text
work/<member>-<short-package>
```

Examples: `work/youssef-linux-ansible`, `work/shaker-wazuh`, `work/iprahim-azure`.

Use small commits with clear messages, for example `docs: record emr01 handoff` or `test: add Gate 3 evidence register`. Do not commit directly to `main`.

## Pull Request checklist

- State the package and gate affected.
- List the exact files changed.
- Mark checklist items `DONE`, `PARTIAL`, `BLOCKED`, or `PENDING`.
- Mark the package state `PENDING`, `IN PROGRESS`, `READY FOR REVIEW`, `PARTIAL`, or `BLOCKED`.
- Mark evidence maturity separately as `NOT RUN`, `STAGED`, `REPRODUCED`, or `MEASURED`.
- Give a sanitised evidence summary, evidence filenames, and measured values, or keep `<PENDING>` when not measured. Raw screenshots and logs stay in the controlled evidence location.
- Describe errors, fixes, dependencies, and any deviation from A-design-package.md.
- Update `team/STATUS.md` in the same PR.
- Confirm that no secret, VM/ISO file, raw evidence, or real patient data is included.

An independent teammate reviews the owner's technical work. Mahmed Shaker checks evidence mapping and report assembly. Mahmed Salh checks sources and data claims. Mahmoud gives final integration sign-off, but does not act as the sole reviewer of his own package. After merge, Mahmoud creates a small status-only PR to mark the package `ACCEPTED` and close its tracking issue. Never mark a package `ACCEPTED` before the work PR is merged.
