# Payload and checksums

The **payload** is the bag of files you carry across the OS boundary. On this kind of machine it is the
*only* route in: C: is BitLocker-encrypted, so "read it from Linux at `/mnt/win/...`" is fiction. The
payload is not a convenience — it is the plan.

## What goes in

| Item | Portable as-is? | Notes |
| --- | --- | --- |
| `memories/` | ✅ | as-is |
| `skills/` | ✅ | content only |
| `cron/jobs.json` | ✅ | **re-path every job** afterwards |
| `config.yaml` | ✅ | diff against the fresh install; let the new platform defaults win |
| `.env` | ✅ | contains **live API keys** — see the warning below |
| `auth.json` | ✅ | |
| `SOUL.md` | ✅ | |
| `sessions/`, `state.db` | ✅ | snapshot `state.db` with the SQLite **backup API** — a plain copy of a live DB can be inconsistent |
| `plugins/`, `desktop-plugins/`, `pets/` | ✅ | |
| Python venvs, `node_modules`, git-installed source, `bin/` launchers | ❌ | **rebuild, don't copy** |

**Snapshot the database properly** (do not copy a live `state.db`, and don't copy `-wal`/`-shm` while
the agent is running):

```bash
python - <<'PY'
import sqlite3
src = sqlite3.connect(r"C:\Users\<you>\AppData\Local\hermes\state.db")
dst = sqlite3.connect("payload/state.db")
with dst:
    src.backup(dst)
PY
```

## The two hard rules

1. **Keep the payload off cloud sync.** It contains live API keys (`.env`, `auth.json`). Put the
   primary copy on the local disk or the shared NTFS volume, `chmod 600` the sensitive files once on
   Linux, and if you keep a copy in a cloud folder, **say what that uploads** before doing it.
2. **Stage it twice: the USB stick *and* a copy on the shared NTFS data partition** (`/mnt/data/hermes-port/`
   in the reference setup). That partition is readable from both OSes — an encrypted C: never is — so
   the stick stops being a single point of failure.

## Build it

```bash
cd /c/Users/<you>
tar -czf D:/hermes-port/hermes-port-payload.tar.gz \
  --exclude='AppData/Local/hermes/**/__pycache__' \
  --exclude='AppData/Local/hermes/**/node_modules' \
  --exclude='AppData/Local/hermes/**/.venv' \
  AppData/Local/hermes
```

**Exclude noisy, rebuildable state:** `logs/`, `audio_cache/`, `browser_cache/`, `.pytest_cache`,
`__pycache__`, `node_modules`, virtualenvs, and (the reference build's mistake) **cron heartbeat/event
files** that change every minute. Shipping those meant the payload was **stale within days** with a
monotonically climbing diff — keep the payload minimal and do a targeted delta re-sync instead of
re-copying the world.

## Checksums and a runbook travel with it

```bash
cd /d/hermes-port
sha256sum hermes-port-payload.tar.gz > SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt          # on the Windows side, before you sign off
```

On the Linux side, after copying and **before** extracting:

```bash
cd ~/hermes-port
sha256sum -c SHA256SUMS.txt
```

For a whole staged tree, `find . -type f -print0 | xargs -0 sha256sum > SHA256SUMS.txt` (the reference
build signed ~2 900 files this way).

**Verify, don't trust** — check the count matches the signed file count.

## Delta re-sync (when the payload is a few days old)

Don't re-copy everything. Diff checksums and move only what changed:

```bash
# Windows: sign the current tree      Linux/target: sign what you have, then compare
comm -23 <(sort new.SHA256SUMS) <(sort old.SHA256SUMS)   # files to send
```

Because the payload **rots** (the DB, sessions, cron run logs all move), re-sign and re-verify at the
moment you carry it, not a week earlier.

Back to [README](../README.md).
