# Nile Care Hospital - Secure Hybrid Enterprise Infrastructure

This repository contains the reviewed design, runbooks, report text, and team integration plan for the Nile Care Hospital graduation project.

## Repository boundaries

- All patient data is synthetic (Synthea) or public anonymised DICOM data.
- Secrets remain in the team vault and are represented by placeholders such as `<SET-IN-VAULT>`.
- VM disks, ISO media, host evidence, archives, historical diffs, and operational simulation scripts remain outside GitHub.
- The published test plan keeps acceptance criteria and evidence requirements. Detailed attack-reproduction procedures stay in the controlled local working copy.

## Layout

- `report/` - design package, runbooks, policies, and report text.
- `team/` - ownership, handoff, integration, and final assembly plan.
- [`team/STATUS.md`](team/STATUS.md) - current work-package status for supervision.
- [`team/09-GITHUB-WORKFLOW.md`](team/09-GITHUB-WORKFLOW.md) - the exact branch, PR, review, and evidence workflow.

## Team delivery workflow

Each member works in a branch named `work/<member>-<package>`, commits only their package, and opens a Pull Request into `main`. The PR must include the completed items, exact files, evidence names, measured results or `<PENDING>`, deviations, and the next dependency. Mahmoud reviews technical consistency and merges accepted work. The status page is updated in the same PR, so a supervisor can see the latest accepted, staged, blocked, and pending work without opening the local VM folders.

The repository is private. Team access and professor read access are granted by GitHub username. Until a person is added as a collaborator, share a reviewed copy or the link only after access is granted; never share GitHub credentials.

Measured results are intentionally left as `<PENDING>` until the corresponding gate and evidence capture are complete.
