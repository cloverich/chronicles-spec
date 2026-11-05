# chronicles-spec history scan

Status: IN PROGRESS (started 2026-11-05 06:30 UTC). Verdict pending.

Scope: every commit reachable from every ref of `cloverich/chronicles-spec` (remote has one ref, `main`, at `68872a2`).

## Stage notes (interim)

- Secrets: gitleaks 8.21.2 over `--all` history: 47 `generic-api-key` hits, all in `vectors/encryption/*.json` and `scripts/encryption-vectors.mjs`. trufflehog 3.82.13 (git + filesystem, no verification): 0. Every key-like field in every historical version of the encryption vectors is a counting byte pattern, all zeros, or HKDF of one; HEAD vectors regenerate byte-identical from the script. Test-only.
- Personal data: one PNG blob in history (1x1, IHDR/IDAT/IEND only; no EXIF/tEXt/iTXt/GPS). Both archive zips hold the same 10 synthetic notes as `vectors/export/basic/expected`, fixed 2024-01-01 timestamps, Unix origin, no comments.
