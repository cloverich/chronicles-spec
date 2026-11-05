# chronicles-spec history scan

**Verdict: GO.** No secrets, no credentials, no personal content anywhere in history. Nothing must be fixed or rewritten before the repo goes public. Two should-fix items are doc edits on `main`. Neither needs a history rewrite. Delete this branch before you change visibility.

Scanned 2026-11-05, 06:29–06:35 UTC. Scope: every object reachable from every ref of `cloverich/chronicles-spec`. The remote has `main` (`68872a2`) and no tags or `refs/pull/*`. That covers 30 commits, one root (`397c1ff`), and 55 distinct paths over time; only `vectors/export/basic/expected/_attachments/pixel.png` has been deleted. Plus this branch.

## Must fix before public

None.

## Should fix

| # | Commit | Path:line | Kind | Suggested fix |
|---|--------|-----------|------|---------------|
| S1 | `b1d5afe` | `AGENTS.md:3` | Private planning pointer: names the Engram journal, a note title, and an Engram note ID (`03h3…`). Public readers can't open it, and it tells them where private planning lives. | Remove the line, or replace it with something neutral ("Planning is tracked outside this repo"). Leave it in history: an ID isn't access. |
| S2 | `b1d5afe` | `AGENTS.md:9-10` | Maintainer workflow that names the private native repo's `Scripts/sync-spec` and the Electron `UPDATE_VECTORS` flow. Not sensitive, but readers can't follow it. | Keep it if contributors are expected; otherwise move it to a `CONTRIBUTING.md` that says the native app is closed. |
| S3 | `b1d5afe` | `README.md:3-4` | Links to `cloverich/chronicles`, which GitHub reports as **private**. Public readers get a 404. (`cloverich/chronicles-electron` is public, so that link works.) | Unlink it ("the native apps, not public") or make it public first. |
| S4 | this branch | `reports/history-scan.md` | This report becomes public along with the repo if the branch still exists. | Delete `audit/history-scan` on the remote after reading. Don't merge it. |

## Owner's call (low)

| # | Commit | Path:line | Kind | Note / fix if wanted |
|---|--------|-----------|------|----------------------|
| O1 | `3b8465d` | `publish.md:30`; `vectors/publish/{basic,missing-attachment,no-title,private-definition}/input.json` (`author`); `vectors/publish/basic/expected/2026-10-14/hello-world-a-first-post/index.md:4` | Owner's first name as example `author` (`"chris"`, already known) | To make it neutral, use `"author": "ada"` and hand-edit the publish vectors in the same commit. |
| O2 | `3b8465d` | `publish.md:123,128`; `vectors/publish/*/input.json` (`name`/`journal` = `pinecoder`) | Owner's personal site and its Hugo `content/blog/` layout as worked examples | Use `example.com` and journal `blog`. Optional, since the domain is the owner's public brand. |
| O3 | `2bd6af5` | `encryption.md:51` | Keychain service `dev.pinecoder.chronicles.library-key` (a reverse-DNS developer identifier) | **Keep.** It's an interop constant that implementations have to match, not a secret, and it ships in the app anyway. |
| O4 | `820b6ca` | `encryption.md:255` | Names Engram as an example delegate | Replace with "a companion app" if Engram should stay unannounced. Otherwise harmless. |
| O5 | 18 of 30 commits | commit trailers | `Co-Authored-By: Claude …` trailers make AI co-authorship public | No action recommended. Rewriting history isn't worth it. |
| O6 | all | commit metadata | Author "Christopher Loverich", email `1010084+cloverich@users.noreply.github.com` (GitHub noreply). No personal email anywhere in history. | None. |
| O7 | `b1d5afe` | `CLAUDE.md:1` | `@AGENTS.md` include for agent tooling | Harmless. Keep it or drop it along with S1. |
| O8 | — | repo root | No `LICENSE` in any commit | Add one before the visibility change; it's already on the checklist. |

Not findings, but a public reader might wonder about them: `encryption.md`/`versions.md` use phase labels E1/E2, which `encryption.md` defines itself, so they're self-explanatory. "Status: draft, implemented by the Swift app only" (`activity.md:5`, `versions.md:4`) is ordinary spec status. Commit subjects mention `spec/…` paths, which `b1d5afe`'s body explains as the earlier home in Electron.

## What was checked

### 1. Secrets and tokens: clean

- **gitleaks 8.21.2** (`git --log-opts=--all --redact`): 47 hits, all `generic-api-key`, all in `vectors/encryption/cases.json`, `vectors/encryption/export.json` (commits `2bd6af5`, `820b6ca`, `febe57a`, `04f31fb`), and `scripts/encryption-vectors.mjs:339` (`b1d5afe`, a test case's `keyId` line).
- **trufflehog 3.82.13** (`git` over full history and `filesystem`, no verification): 0 results.
- **Test keys confirmed synthetic.** I checked every key-like hex field (key/nonce/recovery/bytes/wrap, ≥24 hex chars) in every historical version of both encryption vector files. Each one is a counting byte pattern (`bytes(start, len)`, e.g. `1011…`, `4041…`, `8081…`), all zeros (one deterministic nonce), or HKDF-SHA256 of a counting pattern under a `chronicles.v1 …` info string from the spec. Nothing unexplained: 4 versions, 86 fields. At `HEAD`, `node scripts/encryption-vectors.mjs` in a clean worktree regenerates `vectors/encryption/` byte for byte. The attachment name is SHA-256 of `"test"`.
- Manual pattern search over `git log -p --all` for emails, URLs, IPs, local paths (`/Users/`, `/home/`, `C:\`), iCloud container and team IDs, bundle IDs, and the words password/token/secret found nothing beyond spec prose.

### 2. Personal data: clean

- **Images.** History holds one image blob (`26a8c68`, 68 bytes), the 1×1 "pixel" stored as both `pixel.png` and `_attachments/43/431c…png`. Its PNG chunks are only IHDR/IDAT/IEND: no eXIf, tEXt, iTXt, zTXt, or GPS. The PNG inside both zips is the same blob (SHA-256 `431c…`). `exiftool` isn't installed, so I parsed the chunks directly; no other image format exists in history.
- **Archives.** `vectors/archive/basic/writer.chronicles.zip` and `zip64.chronicles.zip` each have one version in history. Unzipped into an isolated directory, each holds 10 entries matching `vectors/export/basic/expected` plus a fixed `export-info.json`. Every entry is timestamped 2024-01-01 00:00 UTC, created on Unix, with no archive comment. The notes are fixtures ("Plain note", "Frontmatter variety", "Normalization", unicode/emoji, link and attachment edge cases).
- **Vector text.** I pulled all 712 distinct strings from every historical JSON/MD blob under `vectors/` and read them. All are clearly synthetic: shopping lists, "Roadmap follow-up", "Thinking about gardens", search-syntax cases, theme tokens. One encryption plaintext ("Weekend plans", 2026-10-11 timestamp) is a short synthetic sentence about errands. There are no names (other than O1), addresses, phone numbers, health data, or journal-like prose.

### 3–4. Internal references and private planning

Searched every added line in history for: Engram, the owner's name and handles, `pinecoder`, emails, URLs, other `cloverich/*` repos, `chronicles-electron`, `VENDORED`/`sync-spec`, Engram-style IDs in commit bodies, TODO/FIXME/roadmap/beta/App Store/TestFlight/phase/later. All hits are in the tables above or are ordinary spec usage ("defined later", test strings like "roadmap" and "1.0.0-beta"). URLs in vectors are all `example.com`. Commit bodies contain no URLs, emails, or IDs apart from the trailers in O5.

### Limits

- The scan covers what a clone can see. Before going public, GitHub can't show unreachable objects or deleted refs through a clone; there's no sign of force-pushes, and the remote has no PR refs.
- trufflehog ran without verification, so no network calls went to any provider.

## Summary

- **Branch:** `audit/history-scan` (report only; don't merge; delete before going public).
- **Verdict:** GO. No secrets or personal data in any commit, and no history rewrite needed.
- **Do first:** (1) In one doc commit on `main`, remove the Engram pointer from `AGENTS.md` (S1) and fix or remove the private `cloverich/chronicles` link in `README.md` (S3). (2) Add a `LICENSE` (O8). (3) Delete this branch. (4) Optionally neutralize `chris`/`pinecoder` in the publish docs and vectors (O1/O2) and Engram in `encryption.md` (O4).
