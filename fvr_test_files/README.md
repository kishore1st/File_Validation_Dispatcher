# R2 dry-run fixtures (12-case matrix)

One session per case, always through the real Step-3 API (never insert directly).
- Cases 1-3 prove the happy path on the real seed contracts (customer/order/payments).
- Case 8: the FILE is a tiny stub; the 413 comes from the manifest's declared_size (501 MB).
- Case 10: re-upload case01's exact bytes in a NEW session (duplicate / sha256 unique index).
- Case 12: kill one worker pod while case 1 is in VALIDATING; the claim TTL re-drives it.
- Expected outcomes per case: r2_manifest.json; full matrix: 'R2 Checkpoint' notebook cell 2.
