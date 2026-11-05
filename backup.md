# Backups

One backup format and policy for local-first apps (Chronicles, and any other
app that wants it). The conformance surface is
[vectors/backup/manifest.schema.json](vectors/backup/manifest.schema.json):
each app has a test asserting that its emitted manifest validates against it.

The target is any plain directory, typically one managed by a sync service
(iCloud Drive, Google Drive, Dropbox, OneDrive). Snapshots are write-once and
one-way, which is what makes sync folders safe; the live database never goes in
a sync folder.

## Layout

The app owns `<picked>/<app>/` and prunes only inside it. Files it did not
write, including legacy flat backups in `<picked>`, are never matched.

```
<picked>/<app>/
  .lock                                   run lock (O_EXCL, stale after 1 hour)
  snapshots/
    2026-09-22T14-25-34Z/                 createdAt with ':' -> '-'
      manifest.json                       written last
      db.sqlite3
    .tmp-*/                               in-progress; never listed
  attachments/
    <sha256[0:2]>/<sha256><ext>           content-addressed pool
```

A run:

1. Takes the lock; a second concurrent run fails rather than waits.
2. Snapshots the database into `snapshots/.tmp-*/db.sqlite3` with
   `VACUUM INTO ?` (bound parameter), opens it read-only, and requires
   `PRAGMA integrity_check` = `ok` plus app-defined row counts.
3. Hashes every attachment and writes each missing blob via temp file and
   rename. Database first, attachments second: the worst case is an
   unreferenced blob, never a note pointing at a missing file.

   A blob's path is derived from its manifest entry: `<ext>` is the
   extension of `name`, lowercased, kept only if it matches
   `^\.[a-z0-9]{1,10}$`, otherwise empty. The extension makes the pool
   previewable in a file browser; identical bytes under two extensions are
   stored twice. If a bare `<sha256>` from an earlier build exists where
   `<sha256><ext>` is needed, it is renamed, not rewritten; restore falls back
   to the bare name, and GC removes bare blobs nothing references.
4. Writes `manifest.json`, then renames the directory to its UTC stamp.
5. Only after that publish: prunes by retention, deletes invalid snapshot
   directories (no parseable manifest) and stale `.tmp-*`, then garbage-collects
   blobs no surviving manifest references.

## Manifest

```json
{
  "format": 1,
  "app": "chronicles-native",
  "appVersion": "1.0 (412)",
  "createdAt": "2026-09-22T14:25:34Z",
  "trigger": "activity",
  "fingerprint": "<opaque app-defined string>",
  "database": {
    "file": "db.sqlite3",
    "bytes": 11534336,
    "sha256": "…",
    "integrity": "ok",
    "counts": { "notes": 1234, "journals": 5 }
  },
  "attachments": [{ "name": "019040b4-….png", "sha256": "…", "bytes": 48213 }]
}
```

`trigger` is one of `activity`, `manual`, `install`, `pre-restore`. A CLI
invocation is `manual` unless told otherwise (an install script passes
`install`). Counts keys are free-form.

## Triggers

- **Activity:** at startup and hourly while running, snapshot only if the
  newest **regular** snapshot (trigger other than `pre-restore`) is more than
  24 hours old **and** the fingerprint differs from that snapshot's manifest.
  The fingerprint is a cheap query such as `max(updated_at)` plus row counts
  over content tables, so deletes register. It is not wired into write paths.
- **On demand:** manual, install, and pre-restore bypass the check. Manual and
  install snapshots are subject to the same retention; pre-restore snapshots
  are not (see below).

## Retention

A pure function `(snapshot timestamps, policy, now) -> keep set`, applied to
**regular** snapshots only. Default policy: 5 days, 3 weeks, 3 months. All
periods are UTC; weeks are ISO 8601 (Monday start).

- Days tier: group snapshots by calendar day; take the 5 most recent days that
  **have** snapshots; keep the newest snapshot in each.
- Weeks tier: the same over ISO weeks, 3 periods.
- Months tier: the same over calendar months, 3 periods.
- Keep every snapshot taken on `now`'s calendar day.
- Always keep the newest snapshot.

Pre-restore snapshots sit outside the tiers: keep only the most recent one, in
addition to the tiered set, and never count it as "newest" for retention or
the activity check. A pre-restore snapshot often holds the state being
discarded (an empty database after a reset); letting it compete would make it
the day's newest, prune the restore source once the day rolls over, and
suppress activity snapshots for 24 hours. With the exemption, the newest
regular snapshot after a restore is its source, whose fingerprint matches the
live data.

Each app carries a sequence regression test: snapshot, reset, restore, then a
later run on the next day. The restore source must survive, and exactly one
pre-restore snapshot remains.

One snapshot can satisfy several tiers. Quiet stretches never age out the last
good copy, because empty periods are not counted. The result is at most 11
snapshots plus today's.

### Vectors

Every app's retention table test runs these. Default policy unless noted.

| Case                      | now                  | Snapshots                                                                      | Kept                                            |
| ------------------------- | -------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| empty                     | 2026-09-23T12:00:00Z | none                                                                           | none                                            |
| one old snapshot          | 2026-09-23T12:00:00Z | 2020-01-01T00:00:00Z                                                           | it                                              |
| thirty daily              | 2026-09-23T12:00:00Z | 10:00Z daily, 2026-08-25 through 2026-09-23                                    | 08-31, 09-13, 09-19, 09-20, 09-21, 09-22, 09-23 |
| quiet stretch             | 2026-09-23T12:00:00Z | 09:00Z on 2025-11-02, 2025-12-15, 2026-01-10, 02-01, 02-03, 03-05, 03-06       | 2026-01-10, 02-01, 02-03, 03-05, 03-06          |
| everything today survives | 2026-09-23T12:00:00Z | 09-10T08, 09-22T23, 09-23T01, 09-23T05, 09-23T11                               | all five                                        |
| ISO weeks cross the year  | 2026-01-10T12:00:00Z | 12:00Z on 2025-12-28, 2025-12-29, 2026-01-01; policy weeks=2, days=0, months=0 | 2025-12-28, 2026-01-01                          |
| unsorted input            | 2026-09-23T12:00:00Z | 09-01, 09-03, 09-02 (00:00Z); policy days=1, weeks=0, months=0                 | 09-03                                           |

Tiers in the thirty-daily case: 09-23 holds `newest today day week month`,
09-20 holds `day week`, 09-13 holds `week`, and 08-31 holds `month`.

## Restore

1. Verify the chosen snapshot's database (integrity + counts) and stage it
   beside the live database.
2. Take a `pre-restore` snapshot of the current state.
3. Refuse while the live database is open.
4. Atomically rename the staged file over the live database.
5. Materialize the manifest's attachments from the pool: write missing files,
   leave existing files alone, report blobs missing from the pool.

## Host contract

Five verbs in each app's own idiom: `status`, `pickDestination`, `run`,
`list`, `restore`. All logic lives in one module; the rest of the app provides
only a fingerprint query and the attachments directory. A dedicated Backups
page shows destination, last success, last failure with its error, whether
data changed since the last snapshot, the snapshot list (time, age, tier,
size, counts, integrity), and **back up now** / **change destination**.

## Agent boundary

No API, MCP tool, IPC call, or CLI flag reachable by an agent accepts a
destination path. The destination is chosen with a native picker inside the
app and persisted in the app's own config as an opaque handle (a path today, a
security-scoped bookmark once sandboxed). Agents can request a snapshot and
receive names and counts, never contents or a writable path.

The guarantee is narrow: OS grants are per-binary, so an agent with broad disk
access can write to the sync folder regardless. What the app guarantees is that
**it** never performs a write to a path an agent chose.

## Non-goals

Encryption (leave room for a future `.age` artifact), multiple destinations,
and git as an attachment store.
