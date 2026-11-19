# Mahmed Salh - Synthetic Data, Sources, Chapters 1-2, and Compliance Review

ابدأ بجدول مصادر قبل ما تنزّل الداتا. اكتب مين ناشر كل dataset، ترخيصه، واستخدامه بالظبط في EMR أو PACS. خلّي نص الفصول الرسمي بالإنجليزي.

## Mission

Make the report academically defensible: correct data claims, traceable public sources, and clear boundaries between design, staged work, and measured results.

## Primary work

1. Prepare a small Synthea dataset for EMR demonstrations. Record the exact release/source and keep all patients synthetic.
2. Select public anonymised DICOM data for Orthanc. Record source URL, dataset name, licence/terms, download date, and hash if available. Do not describe DICOM as Synthea output.
3. Own Chapter 1 using E-report-text.md and draft Chapter 2 from A §§9 and 12, the design and the official template: background, problem, objectives, scope, requirements, assets, risks, and legal framing.
4. Review references in E and D for real URLs and accurate statutory wording. Keep the official regulation citation pending if it cannot be verified.
5. Review D-policies.md for policy-to-control-to-evidence links and synthetic-data wording.

## Required outputs

- `HANDOFF-Mahmed-Salh.md`.
- `data-manifest.md` with Synthea/DICOM source, version, licence, hash, and intended use.
- Synthetic sample files or a reproducible generation command; never real patient data.
- Revised Chapter 1/2 text and an IEEE-style reference list with stable URLs and access dates where appropriate.
- Compliance review log listing each claim checked, source, and any `<PENDING>` item.

## Acceptance checks

- No sentence says all clinical data came from Synthea; EMR records and public anonymised DICOM are separated.
- No RTO, RPO, Nessus, Lynis, AppLocker, FSRM, or restore number is invented.
- Every legal claim points to a real source or is explicitly marked pending verification.
- Data files contain no real personal data and no credentials.
- Mahmed Shaker receives the final source list for the References section and Mahmoud receives any design-risk finding.

## Where your work goes

Give Youssef the data manifest, the small batches for EMR/PACS, and expected counts based on the files actually handed off. He reports the observed import counts after validation. Give Shaker your Chapter 1 and 2 drafts, references, declaration wording and comments on Chapter 10. Use [the handoff template](08-HANDOFF-TEMPLATE.md) for each dataset, each chapter, and the policy and reference review.
