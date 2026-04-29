# rclone — From Beginner to Expert

A hands-on, lesson-by-lesson course for **rclone**, the Swiss-army knife for cloud storage.
Every lesson has a **Goal**, **Concept**, **Hands-on** commands you can run, and an **Exercise**.

> Convention: lines starting with `$` are commands you type. Lines without `$` are example output.
> Replace `gdrive`, `s3`, `backup`, etc. with your own remote names.
> Always `--dry-run` first when you're not sure. Seriously.

---

## Table of Contents

**Beginner**
1. [What is rclone? Installation](#lesson-1--what-is-rclone--installation)
2. [Your first remote](#lesson-2--your-first-remote)
3. [Listing files](#lesson-3--listing-files)
4. [Copying files](#lesson-4--copying-files)
5. [Syncing files (with dry-run safety)](#lesson-5--syncing-files-with-dry-run-safety)

**Intermediate**

6. [Working with multiple remotes](#lesson-6--working-with-multiple-remotes)
7. [Filtering: include / exclude / filter-from](#lesson-7--filtering-include--exclude--filter-from)
8. [Encryption with the `crypt` remote](#lesson-8--encryption-with-the-crypt-remote)
9. [Mounting a remote as a filesystem](#lesson-9--mounting-a-remote-as-a-filesystem)
10. [Configuration management & secrets](#lesson-10--configuration-management--secrets)

**Advanced**

11. [Performance: transfers, checkers, bandwidth](#lesson-11--performance-transfers-checkers-bandwidth)
12. [Server-side copy & move (cloud-to-cloud)](#lesson-12--server-side-copy--move-cloud-to-cloud)
13. [Two-way sync with `bisync`](#lesson-13--two-way-sync-with-bisync)
14. [Automation: cron, systemd, scripts](#lesson-14--automation-cron-systemd-scripts)
15. [Troubleshooting & log analysis](#lesson-15--troubleshooting--log-analysis)

**Expert**

16. [`rclone serve`: HTTP, WebDAV, SFTP, S3](#lesson-16--rclone-serve-http-webdav-sftp-s3)
17. [Union remote: merging backends](#lesson-17--union-remote-merging-backends)
18. [Chunker remote: splitting huge files](#lesson-18--chunker-remote-splitting-huge-files)
19. [Backend quirks: S3, Dropbox, OneDrive, Google Drive](#lesson-19--backend-quirks-s3-dropbox-onedrive-google-drive)
20. [The RC (Remote Control) API & building from source](#lesson-20--the-rc-remote-control-api--building-from-source)

[Cheat Sheet](#cheat-sheet) · [Glossary](#glossary)

---

# Beginner

## Lesson 1 — What is rclone & installation

**Goal:** Install rclone and understand what problem it solves.

**Concept.** rclone is a single static binary that talks to **70+ cloud and protocol backends** (S3, Google Drive, Dropbox, OneDrive, Backblaze B2, SFTP, WebDAV, local disk, etc.) using one consistent CLI. Think of it as `rsync` for the cloud, plus a mounter, plus a server, plus an encryption layer.

**Hands-on.**

```bash
# macOS
$ brew install rclone

# Linux / WSL (official installer)
$ curl https://rclone.org/install.sh | sudo bash

# Windows (PowerShell, scoop)
$ scoop install rclone

# Verify
$ rclone version
rclone v1.xx.x
```

Get help:

```bash
$ rclone help            # global help
$ rclone help commands   # list every command
$ rclone copy --help     # help for one command
```

**Exercise.**
1. Install rclone, run `rclone version`, write down the version.
2. Run `rclone help backends` and pick three backends you might actually use. Why those?

---

## Lesson 2 — Your first remote

**Goal:** Configure two remotes — a local one and a cloud one — using `rclone config`.

**Concept.** A **remote** is a named connection to a backend, stored in `rclone.conf`. Syntax everywhere is `remote:path`. `local:` is implicit — you can also reference local paths directly (`./data`, `/Users/you/Pictures`).

**Hands-on — interactive config.**

```bash
$ rclone config
# Choose: n) New remote
# Name: gdrive
# Storage: drive          (Google Drive)
# client_id / client_secret: leave blank for now (uses rclone's defaults — slower)
# scope: 1 (full access)  or 2 (read-only)
# Then a browser opens for OAuth — log in and approve.
# Save and quit.
```

You can also create a non-cloud "alias" remote that just points at a local folder:

```bash
$ rclone config create backup alias remote=/Users/$USER/Backups
```

Inspect what you've created:

```bash
$ rclone listremotes
gdrive:
backup:

$ rclone config show gdrive
```

**Exercise.**
1. Configure one cloud remote (Google Drive, Dropbox, OneDrive, or an S3 bucket — any will do).
2. Configure an `alias` remote called `home` that points at your home directory.
3. Run `rclone about gdrive:` — what does it tell you?

---

## Lesson 3 — Listing files

**Goal:** Look around remotes the way you'd `ls` a directory.

**Concept.** rclone has several listing flavors because cloud storage is messy:

| Command   | Shows                                |
|-----------|--------------------------------------|
| `lsd`     | Directories only                     |
| `ls`      | Files (recursive) with size          |
| `lsl`     | Files with size + modtime            |
| `lsf`     | Plain file paths (great for piping)  |
| `tree`    | A tree view                          |
| `size`    | Total size + file count              |
| `ncdu`    | Interactive disk-usage TUI           |

**Hands-on.**

```bash
$ rclone lsd gdrive:
$ rclone ls gdrive:Photos | head
$ rclone lsl gdrive:Photos/2025
$ rclone size gdrive:
Total objects: 12,345
Total size:    240.123 GiB

# Pipe-friendly output
$ rclone lsf -R --files-only gdrive:Documents > all_docs.txt

# Interactive — like ncdu for the cloud
$ rclone ncdu gdrive:
```

**Exercise.**
1. Find the **largest folder** in your cloud remote using `rclone ncdu`.
2. Use `rclone lsf -R` to count how many `.pdf` files you have:
   ```bash
   $ rclone lsf -R --include "*.pdf" gdrive: | wc -l
   ```

---

## Lesson 4 — Copying files

**Goal:** Move data between local and cloud (or cloud and cloud).

**Concept.** `rclone copy` is **non-destructive**: it copies missing or changed files, but never deletes anything at the destination. It compares by **size + modtime** by default, optionally by hash.

**Hands-on.**

```bash
# Local → cloud (whole folder)
$ rclone copy ./photos gdrive:Photos -P
#                                     ^ -P = live progress

# Cloud → local
$ rclone copy gdrive:Photos/2025 ./photos-2025 -P

# Single file: --copy-links to follow symlinks, --no-traverse to skip listing dest
$ rclone copyto ./report.pdf gdrive:Documents/report-2026.pdf
```

Useful flags:

- `-P` / `--progress`  live progress
- `-v` / `-vv`  verbose / debug
- `--checksum`  compare by hash, not modtime (slower but bulletproof)
- `--ignore-existing`  skip if name exists at dest (don't even check size)
- `--max-size 100M` / `--min-size 10K`  size filters

**Exercise.**
1. Copy a folder up with `-P` and watch the throughput.
2. Re-run the same command — observe that **no bytes transfer** the second time. Why?
3. Touch a file (`touch ./photos/foo.jpg`) and re-run with `-vv`. What does rclone log?

---

## Lesson 5 — Syncing files (with dry-run safety)

**Goal:** Make a destination **identical** to a source — including deletions.

**Concept.** `rclone sync` is `copy` + **deletes anything at the destination that isn't at the source**. It's powerful and dangerous. **Always `--dry-run` first.**

```
copy   = add + update      (safe)
sync   = add + update + DELETE  (dangerous — irreversible without --backup-dir)
move   = copy + delete source
```

**Hands-on.**

```bash
# 1) Always dry-run first
$ rclone sync ./photos gdrive:Photos --dry-run -v

# 2) If output looks right, run for real with progress
$ rclone sync ./photos gdrive:Photos -P

# 3) Safer sync: keep "deleted" files in a dated trash folder instead of removing them
$ rclone sync ./photos gdrive:Photos \
    --backup-dir gdrive:Photos-trash/$(date +%F) \
    -P
```

Other safety nets:

- `--max-delete 100` — abort if more than 100 files would be deleted.
- `--track-renames` — detect renames so a renamed file doesn't re-upload.
- `--immutable` — refuse to modify or delete anything (useful for archives).

**Exercise.**
1. Create a `./test-src` and `./test-dst` folder, populate `test-src` with 5 files, sync them.
2. Delete one file from `test-src`, run `rclone sync --dry-run` — predict the output before reading it.
3. Re-run with `--backup-dir`. Where did the "deleted" file go?

---

# Intermediate

## Lesson 6 — Working with multiple remotes

**Goal:** Treat 3+ remotes as one toolkit.

**Concept.** Once you have multiple remotes, `rclone` becomes a universal data router. Any command that takes `src dst` works **between any two remotes** — local↔cloud, cloud↔cloud, even cloud↔chained-remote.

**Hands-on.**

```bash
$ rclone listremotes
gdrive:
s3:
backup:
home:

# Cloud → cloud directly (often server-side, no local bandwidth — see Lesson 12)
$ rclone copy gdrive:Photos s3:my-bucket/photos -P

# Compare two remotes for differences
$ rclone check gdrive:Photos s3:my-bucket/photos --one-way

# Equivalent of "rclone diff"
$ rclone check src: dst: --differ differ.txt --missing-on-dst missing.txt
```

`rclone check` modes:

- default: compare by **hash** (fastest if both backends have the same hash type).
- `--size-only`: cheap, less reliable.
- `--download`: hash both sides locally — last resort.

**Exercise.**
1. Add a second cloud remote.
2. Use `rclone check` to compare the same folder on both. Are the hashes compatible?
3. If `--download` is needed, that's a clue your two backends use **different hash types**. Which?

---

## Lesson 7 — Filtering: include / exclude / filter-from

**Goal:** Move only the files you actually want.

**Concept.** rclone evaluates filter rules **in order**, first match wins. The two main forms:

```
+ pattern    # include
- pattern    # exclude
```

Plus convenience flags `--include`, `--exclude`, `--include-from`, `--exclude-from`, and the all-in-one `--filter-from`.

Pattern syntax (glob-like):

| Pattern             | Matches                              |
|---------------------|--------------------------------------|
| `*.jpg`             | `.jpg` in any dir                    |
| `**.jpg`            | same — `**` = any depth              |
| `/secret/**`        | everything under top-level `secret/` |
| `*.{tmp,bak}`       | brace alternation                    |
| `**/.DS_Store`      | macOS junk anywhere                  |

**Hands-on.**

```bash
# Quick one-liner: only photos and videos
$ rclone copy ./Pictures gdrive:Pics \
    --include "*.{jpg,jpeg,png,heic,mp4,mov}" -P

# Exclude noise
$ rclone sync ./project gdrive:project \
    --exclude ".git/**" --exclude "node_modules/**" --exclude "*.log" -P

# Bigger filter set in a file
$ cat > backup.filter <<'EOF'
- .DS_Store
- ._*
- node_modules/**
- .git/**
+ *.{md,py,js,ts,go,rs}
+ docs/**
- *
EOF

$ rclone sync ./code gdrive:code-backup --filter-from backup.filter -P --dry-run
```

> The trailing `- *` in a `+ ... + ... - *` filter means "everything not explicitly included is excluded".

**Exercise.**
1. Build a filter file that backs up `~/Documents` but skips `Downloads`, anything `>500MB`, and any `.iso`.
   - Hint: combine `--filter-from` with `--max-size 500M`.
2. Run with `--dry-run -v` and verify the file list before running it for real.

---

## Lesson 8 — Encryption with the `crypt` remote

**Goal:** Store data **encrypted client-side** so the cloud provider can't read it.

**Concept.** `crypt` is a *wrapper* remote: it sits in front of an existing remote and encrypts file contents and (optionally) names. Two passwords are used — both should be stored in your config (and the config itself should be password-protected, see Lesson 10).

**Hands-on.**

```bash
$ rclone config
# n) New remote
# name: secret
# storage: crypt
# remote: gdrive:Encrypted   <-- where the encrypted blobs live
# filename_encryption: standard   (or off / obfuscate)
# directory_name_encryption: true
# password: <generate a strong one — rclone can do it for you>
# password2 (salt): <generate another>
# Save.
```

Use it like any other remote — files are transparently encrypted on upload, decrypted on download:

```bash
$ rclone copy ./tax-2025 secret: -P
$ rclone ls secret:
$ rclone lsd gdrive:Encrypted   # you'll see scrambled names — that's the point
```

> **Lose the passwords → lose the data.** Back up your `rclone.conf` (it contains the obscured keys) somewhere safe, like a password manager.

**Exercise.**
1. Create a `crypt` remote on top of a folder in your cloud remote.
2. Copy a small text file in. Open the cloud's web UI — confirm you can't read the contents or names.
3. `rclone cat secret:hello.txt` — confirm decryption works.

---

## Lesson 9 — Mounting a remote as a filesystem

**Goal:** Use a cloud remote like a regular folder via `mount`.

**Concept.** `rclone mount` exposes a remote through **FUSE** (macFUSE on macOS, WinFsp on Windows, fuse3 on Linux). Apps see normal files; rclone fetches/uploads on demand and caches locally.

**Hands-on.**

```bash
$ mkdir -p ~/mnt/gdrive

# Foreground — Ctrl-C to unmount
$ rclone mount gdrive: ~/mnt/gdrive \
    --vfs-cache-mode full \
    --vfs-cache-max-size 10G \
    --dir-cache-time 1h \
    --poll-interval 15s \
    -vv

# In another terminal:
$ ls ~/mnt/gdrive
$ cp big.iso ~/mnt/gdrive/Backups/    # uploads in the background

# Unmount
$ umount ~/mnt/gdrive          # macOS / Linux
$ fusermount -u ~/mnt/gdrive   # Linux (alternative)
```

**Cache modes** (from least to most magic):

- `off` — read-through only, writes fail unless whole file fits in memory. Avoid.
- `minimal` — cache file headers.
- `writes` — buffer writes, good for sequential uploads.
- `full` — full read+write cache. **Use this** unless you have a reason not to.

**Exercise.**
1. Mount a remote with `--vfs-cache-mode full` and copy a 100MB+ file in. Watch the cache directory (`~/.cache/rclone/vfs/...`) grow.
2. Edit a file in place (e.g. `vim ~/mnt/gdrive/notes.md`). Does it work? With which cache mode?
3. Unmount cleanly. What flags would you add to mount it on boot via systemd?

---

## Lesson 10 — Configuration management & secrets

**Goal:** Keep your `rclone.conf` portable, encrypted, and scriptable.

**Concept.** The config file (`rclone config file` to find it — usually `~/.config/rclone/rclone.conf`) holds OAuth tokens and (obscured) passwords. You can:

- **Encrypt it** at rest with a master password.
- **Override settings** per-command with `--<backend>-<flag>` or environment variables.
- **Provision** entirely from env vars (no config file at all) — perfect for CI/Docker.

**Hands-on.**

```bash
# Encrypt the config with a master password
$ rclone config
#  s) Set configuration password

# Now every command needs the password. Provide it via env so scripts work:
$ export RCLONE_CONFIG_PASS='your-strong-password'

# Inspect / find the config
$ rclone config file
$ rclone config show
$ rclone config redacted   # safe to share — tokens & passwords removed

# CLI override (no edit to rclone.conf)
$ rclone --s3-access-key-id=AKIA... --s3-secret-access-key=... \
    --s3-endpoint=https://s3.amazonaws.com \
    ls :s3:my-bucket          # the leading ":" = "ad-hoc unnamed remote"

# Same thing via env vars (great for Docker / CI)
$ RCLONE_CONFIG_S3_TYPE=s3 \
  RCLONE_CONFIG_S3_PROVIDER=AWS \
  RCLONE_CONFIG_S3_ACCESS_KEY_ID=AKIA... \
  RCLONE_CONFIG_S3_SECRET_ACCESS_KEY=... \
  rclone ls s3:my-bucket
```

**Exercise.**
1. Encrypt your config. Make sure scripts still work using `RCLONE_CONFIG_PASS`.
2. Use `rclone config redacted > rclone.safe.conf` and check what's stripped.
3. Provision a remote **only** via env vars and confirm `rclone ls` works without touching `rclone.conf`.

---

# Advanced

## Lesson 11 — Performance: transfers, checkers, bandwidth

**Goal:** Make rclone go fast (or politely slow).

**Concept.** Three knobs dominate transfer speed:

| Flag                   | What it controls                                       | Sensible starting point |
|------------------------|--------------------------------------------------------|--------------------------|
| `--transfers N`        | Parallel **file** transfers                            | 4–16 (default 4)         |
| `--checkers N`         | Parallel "does this file exist?" checks                | 8–32 (default 8)         |
| `--multi-thread-streams N` | Parallel streams **per file** (big files)          | 4–8                      |
| `--bwlimit RATE`       | Cap bandwidth — supports schedule strings              | `8M` or `08:00,1M 18:00,off` |
| `--buffer-size`        | In-memory buffer per transfer                          | 16M–64M                  |
| `--use-mmap`           | Use mmap for buffers (lower mem)                       | flag (no value)          |

**Hands-on.**

```bash
# Saturate a fast pipe
$ rclone copy s3:big-bucket ./local -P \
    --transfers 16 --checkers 32 \
    --multi-thread-streams 8 \
    --buffer-size 32M --use-mmap

# Be a polite neighbor — full speed at night, throttled during the day
$ rclone sync ./photos gdrive:Photos \
    --bwlimit "08:00,1M 18:00,off" -P
#           ^ 1 MB/s 08:00–18:00, unlimited otherwise

# Hit a remote in dry-run to benchmark listing speed
$ time rclone size gdrive:bigtree --fast-list
```

`--fast-list` does **one big listing** instead of recursing — much faster on S3/Drive when you have many files, but uses more RAM.

**Exercise.**
1. Time the same copy with `--transfers 4` vs `--transfers 16`. When does extra parallelism stop helping?
2. Use `--bwlimit` to cap a sync at 1M. Watch with `iftop`/Activity Monitor — does it really stay under?

---

## Lesson 12 — Server-side copy & move (cloud-to-cloud)

**Goal:** Move files **between two locations on the same provider** without using your bandwidth.

**Concept.** Many backends (S3, Drive, Dropbox, B2, OneDrive) can copy data internally. rclone uses this automatically when both `src` and `dst` are on the **same remote** (or two remotes pointing at the same provider). You can also force the issue across two remotes of the same provider type.

**Hands-on.**

```bash
# Same remote → server-side automatically
$ rclone copy gdrive:Photos/2024 gdrive:Archive/2024 -P
# In -vv logs: "Copied (server-side copy)"

# Two remotes, both Google Drive accounts → also server-side
$ rclone copy gdrive-personal:Stuff gdrive-work:Stuff --drive-server-side-across-configs -P

# S3 bucket-to-bucket within AWS
$ rclone copy s3:bucket-a/path s3:bucket-b/path -P
```

When server-side **isn't** available (e.g. Dropbox → S3), rclone falls back to streaming through your machine. To detect that, watch the bandwidth indicator in `-P`: if it's nonzero, data is going through you.

**Exercise.**
1. Server-side copy a folder within your Drive — confirm via `-vv` logs.
2. Try a copy across two **different providers** — observe local bandwidth being used.

---

## Lesson 13 — Two-way sync with `bisync`

**Goal:** Keep two locations in sync **bidirectionally** (laptop ↔ cloud).

**Concept.** `bisync` records the state of both sides at last sync, detects changes on each, and propagates them. First run is a "resync" baseline.

**Hands-on.**

```bash
# 1) First run establishes a baseline (no sync yet)
$ rclone bisync ./Notes gdrive:Notes --resync -P

# 2) Subsequent runs propagate changes both directions
$ rclone bisync ./Notes gdrive:Notes -P

# Conflicts are renamed with .conflict1 / .conflict2 by default
# Force one side to win on conflict:
$ rclone bisync ./Notes gdrive:Notes --conflict-resolve newer -P
```

Recommended flags for real use:

- `--check-access` — abort if a sentinel file (`RCLONE_TEST`) is missing on either side. Prevents nuking a wrongly-mounted remote.
- `--max-delete 25` — abort if too many deletions detected.
- `--filters-file my.filter` — share filters across runs.

**Exercise.**
1. `bisync --resync` a small folder.
2. Edit a file on each side **without syncing in between**, then run bisync. Inspect the `.conflict*` files.
3. Add a `RCLONE_TEST` file to both sides and re-run with `--check-access`.

---

## Lesson 14 — Automation: cron, systemd, scripts

**Goal:** Run rclone unattended, with logs and alerts.

**Hands-on — cron (macOS/Linux).**

```bash
$ crontab -e
# Backup every night at 02:30
30 2 * * *  /usr/local/bin/rclone sync /Users/me/Documents gdrive:Backups/Documents \
              --backup-dir gdrive:Backups/_trash/$(date +\%F) \
              --log-file /Users/me/.rclone/sync.log --log-level INFO \
              --max-delete 200
```

Tips:

- Use **absolute paths** to `rclone` and to all source/dest folders.
- `%` in cron is special — escape as `\%`.
- Pipe stderr to a log so you can debug failures: `--log-file ... --log-level INFO`.
- Use `flock` to prevent overlapping runs:
  ```bash
  flock -n /tmp/rclone.lock rclone sync ...
  ```

**Hands-on — systemd timer (Linux).**

```ini
# ~/.config/systemd/user/rclone-backup.service
[Unit]
Description=rclone nightly backup

[Service]
Type=oneshot
ExecStart=/usr/bin/rclone sync %h/Documents gdrive:Backups/Documents \
  --backup-dir gdrive:Backups/_trash/%Y-%m-%d \
  --log-file %h/.rclone/sync.log --log-level INFO

# ~/.config/systemd/user/rclone-backup.timer
[Unit]
Description=Run rclone backup nightly

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
$ systemctl --user daemon-reload
$ systemctl --user enable --now rclone-backup.timer
$ systemctl --user list-timers
```

**Exercise.**
1. Schedule a nightly sync that writes logs to `~/.rclone/sync.log` and rotates them weekly (use `logrotate` or the macOS equivalent).
2. Add a `--max-delete` safety so a runaway sync can't wipe your archive.

---

## Lesson 15 — Troubleshooting & log analysis

**Goal:** When things go wrong, find out *why* fast.

**Toolbox.**

```bash
# 1) Crank up verbosity
$ rclone copy ... -vv --log-file rclone.log --log-level DEBUG

# 2) Validate transfers without doing them
$ rclone check src: dst: --one-way

# 3) Force re-hash a single object
$ rclone hashsum md5 gdrive:bigfile.iso

# 4) See *exactly* what the API returned
$ rclone copy ... --dump headers   # or --dump bodies (huge!)
```

**Common pitfalls**

| Symptom                                               | Likely cause / fix                                                                 |
|-------------------------------------------------------|-------------------------------------------------------------------------------------|
| "Failed to copy: Forbidden"                           | Wrong scope or expired token. `rclone config reconnect remote:`                     |
| Same file re-uploads every run                        | modtime not preserved by backend. Add `--checksum` or `--update --use-server-modtime` |
| 403 / quotaExceeded on Google Drive                   | API limit. Add your **own** `client_id`/`client_secret` in `rclone config`          |
| sync wiped files unexpectedly                         | Used `sync` not `copy`. Always `--dry-run` and `--max-delete N` for sync            |
| Mount feels slow                                      | Wrong cache mode. Use `--vfs-cache-mode full`                                       |
| "directory not empty" on delete                       | Hidden `.DS_Store` or `Thumbs.db`. Add to `--exclude`                               |

**Exercise.**
1. Run a copy with `-vv --dump headers` against a tiny file. Read one request/response pair end-to-end.
2. Force a controlled failure (rename a remote in mid-flight) and read the resulting log. Identify the first ERROR line.

---

# Expert

## Lesson 16 — `rclone serve`: HTTP, WebDAV, SFTP, S3

**Goal:** Turn any remote into a server other tools can talk to.

**Concept.** `rclone serve <protocol>` exposes a remote (or part of it) over a chosen protocol. Combine with `crypt` and you have a self-hosted, encrypted, multi-backend cloud share.

**Hands-on.**

```bash
# Read-only HTTP browse on http://localhost:8080
$ rclone serve http gdrive:Public --addr :8080

# WebDAV — mount in Finder/Explorer/macOS as a network drive
$ rclone serve webdav gdrive: --addr :8081 \
    --user me --pass 'secret'

# SFTP — feels like a real SSH server backed by S3
$ rclone serve sftp s3:my-bucket --addr :2022 \
    --user me --pass 'secret' --authorized-keys ~/.ssh/authorized_keys

# Full S3 API gateway — point any S3 SDK at rclone
$ rclone serve s3 gdrive:Backups --addr :8000 \
    --auth-key access_key_id,secret_access_key
```

Combined with `--vfs-cache-mode full`, these servers behave well even for random access patterns.

**Exercise.**
1. `rclone serve webdav` your Drive on `:8081` and mount it from Finder (`cmd-K → http://localhost:8081`).
2. `rclone serve s3` over a `crypt` remote and copy into it using the AWS CLI:
   ```bash
   $ aws --endpoint-url http://localhost:8000 s3 cp ./file.txt s3://default/
   ```

---

## Lesson 17 — Union remote: merging backends

**Goal:** Treat several remotes as **one** filesystem with policies for where new files land.

**Concept.** A `union` remote stacks "upstreams" with **create**, **action**, and **search** policies (similar to mergerfs). New uploads land on the upstream chosen by the create policy; reads walk all upstreams.

**Hands-on.**

```bash
$ rclone config
# n) name: pool
# storage: union
# upstreams: gdrive: s3: dropbox:        (space separated)
# action_policy:  epall   (act on all that have the file)
# create_policy:  ff      (first-found upstream that's writable)
# search_policy:  ff
```

Useful upstream tags:

- `gdrive:Hot:rw` — read+write
- `s3:Cold:ro` — read-only (archive tier)
- `local:/srv/cache:nc` — no-create (reads only fall through)

```bash
$ rclone lsd pool:
$ rclone copy ./newfile.txt pool:Inbox -P  # lands on the first writable upstream
```

**Exercise.**
1. Build a `pool` of two remotes: a local SSD (`local:/srv/hot`) and a cloud archive (`gdrive:Cold:ro`).
2. Use the `lus` (least-used-space) create policy to balance writes across multiple writable upstreams.

---

## Lesson 18 — Chunker remote: splitting huge files

**Goal:** Store files larger than your backend's per-object limit, transparently.

**Concept.** `chunker` is another wrapper remote. It splits big uploads into N-sized parts on the underlying remote, and reassembles them on read. Crucial for backends with 5 GB or 50 GB caps.

**Hands-on.**

```bash
$ rclone config
# n) name: bigfiles
# storage: chunker
# remote: gdrive:Chunks
# chunk_size: 1G               (good default)
# hash_type: md5
# transactions: rename         (safest)

# Now upload an arbitrary large file — rclone splits it transparently
$ rclone copy ./vm-disk.qcow2 bigfiles:VMs/ -P

# On gdrive:Chunks you'll see:
#   vm-disk.qcow2.rclone_chunk.001
#   vm-disk.qcow2.rclone_chunk.002
#   ...
# Through bigfiles: it appears as one file.
```

You can stack: `chunker` over `crypt` over `gdrive` = encrypted, chunked, cloud storage.

**Exercise.**
1. Build a `chunker` over a remote with a small `--chunk-size` (e.g. 8M). Upload a 50MB file. Inspect the underlying remote.
2. Stack `chunker → crypt → s3`. Verify decryption + reassembly work end-to-end.

---

## Lesson 19 — Backend quirks: S3, Dropbox, OneDrive, Google Drive

**Goal:** Know the gotchas that bite everyone eventually.

**S3 (and clones — Backblaze B2, Cloudflare R2, MinIO, Wasabi).**

- Set the right `provider` (`AWS`, `Cloudflare`, `Backblaze`, `Other`) — it changes signatures.
- `--s3-no-check-bucket` skips a permissions probe — useful when your IAM only allows specific prefixes.
- Storage classes: `--s3-storage-class STANDARD_IA|GLACIER|DEEP_ARCHIVE` for cheap archives. Watch retrieval costs.
- Use `--fast-list` for huge buckets; use `--s3-list-version 2` if your provider supports it.

**Google Drive.**

- The default OAuth client is **shared and rate-limited**. Always create your own `client_id`/`client_secret` for serious use — see the rclone docs for "Making your own client_id".
- Drive doesn't store MD5/SHA — rclone uses Drive's own checksum. Cross-backend `check` may need `--download`.
- Shared Drives: set `team_drive` ID in config (or `--drive-team-drive`).
- `--drive-acknowledge-abuse` to download files Google flagged.

**Dropbox.**

- Has weird path constraints (case-insensitive, certain chars forbidden). rclone handles most of this; use `--dropbox-batch-mode async` for many small files.
- 350 GB upload session limit per file — combine with `chunker` if you exceed it.

**OneDrive.**

- Throttling is aggressive. Lower `--transfers` (try 4) if you hit `429`.
- Personal vs Business uses different endpoints — `rclone config` asks which.
- Hash type is `QuickXorHash` (Business) or `SHA1` (Personal). Cross-backend `check` will need `--download`.

**Exercise.**
1. Pick the cloud you actually use. Read its dedicated page on rclone.org and write down the **three flags** you'd add to your standard `sync` for that backend.
2. Demonstrate the difference: same `rclone check` command, once with default hashing and once with `--download`. When does it matter?

---

## Lesson 20 — The RC (Remote Control) API & building from source

**Goal:** Drive rclone programmatically; tweak the source.

**Concept.** rclone embeds an HTTP **Remote Control API**. Start a daemon, then issue commands via HTTP/JSON — perfect for dashboards, schedulers, or other programs.

**Hands-on — start the daemon.**

```bash
$ rclone rcd --rc-web-gui --rc-addr :5572 --rc-user me --rc-pass secret
# Opens a browser GUI; or call the API directly:

$ curl -u me:secret -X POST http://localhost:5572/core/stats
{"bytes":0,"checks":0,"deletes":0,"elapsedTime":1.2, ... }

$ curl -u me:secret -X POST http://localhost:5572/sync/copy \
    -d srcFs=gdrive:Photos -d dstFs=s3:photos-backup \
    -d _async=true
{"jobid":42}

$ curl -u me:secret -X POST http://localhost:5572/job/status -d jobid=42
```

Useful endpoints:

- `core/stats`, `core/bwlimit`, `core/memstats`
- `operations/list`, `operations/copyfile`, `operations/movefile`
- `sync/sync`, `sync/copy`, `sync/move`, `sync/bisync`
- `job/list`, `job/status`, `job/stop`
- `vfs/refresh`, `mount/mount`, `mount/listmounts`

**Build from source.**

```bash
$ git clone https://github.com/rclone/rclone
$ cd rclone
$ go build              # produces ./rclone
$ ./rclone version      # confirm your build
```

To experiment, look at:

- `cmd/`         — every command (`copy`, `sync`, `mount`, ...) lives in its own subpackage.
- `backend/`     — one folder per backend. Adding a backend is the canonical "advanced" PR.
- `fs/sync/`     — the sync engine.
- `vfs/`         — the FUSE-facing virtual filesystem.

**Exercise.**
1. Start `rcd` and run a copy via the HTTP API with `_async=true`. Poll `job/status` until it finishes.
2. Build rclone from source. Add a `Println` somewhere harmless (e.g. start of `cmd/copy/copy.go`) and watch it appear when you run your build.
3. (Stretch) Look at `backend/local/local.go` and identify how `Move` falls back to "copy + delete" when crossing filesystems.

---

# Cheat Sheet

```text
# Discover
rclone listremotes
rclone about REMOTE:
rclone size REMOTE:PATH
rclone tree REMOTE:PATH
rclone ncdu REMOTE:PATH

# Move data
rclone copy   SRC DST -P              # add/update, never deletes
rclone sync   SRC DST -P --dry-run    # mirror; ALWAYS dry-run first
rclone move   SRC DST -P              # copy then delete src
rclone copyto SRC DST                 # single-file copy with rename

# Verify
rclone check  SRC DST --one-way
rclone hashsum md5|sha1|sha256 REMOTE:FILE

# Filter
--include "*.jpg"  --exclude ".git/**"  --filter-from file.txt
--max-size 1G  --min-size 10K  --max-age 30d

# Speed
--transfers 8 --checkers 16 --multi-thread-streams 8
--buffer-size 32M --use-mmap --fast-list
--bwlimit "08:00,1M 18:00,off"

# Safety
--dry-run -v
--backup-dir REMOTE:trash/$(date +%F)
--max-delete 100
--immutable

# Mount
rclone mount REMOTE: ~/mnt/X --vfs-cache-mode full --dir-cache-time 1h

# Serve
rclone serve {http|webdav|sftp|s3|ftp|nfs} REMOTE: --addr :PORT

# Daemon / API
rclone rcd --rc-web-gui --rc-addr :5572
```

# Glossary

- **Remote** — a named connection to a backend (local, cloud, SFTP, etc.).
- **Backend** — a protocol implementation (S3, Drive, WebDAV, ...).
- **Wrapper remote** — a remote that sits in front of another (`crypt`, `chunker`, `union`, `alias`).
- **Server-side copy** — when source and destination are on the same provider and the data never touches your machine.
- **VFS** — rclone's virtual filesystem layer used by `mount` and `serve`.
- **RC** — the embedded HTTP "remote control" API.
- **Bisync** — bidirectional sync that records state on both sides.
- **`--dry-run`** — your best friend. Print what *would* happen, do nothing.

---

## Where to go next

- Official docs: <https://rclone.org/docs/>
- Per-backend pages: <https://rclone.org/overview/>
- Source: <https://github.com/rclone/rclone>
- Forum (great for backend-specific weirdness): <https://forum.rclone.org/>

Happy syncing — and remember: **`--dry-run` first.**
