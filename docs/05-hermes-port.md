# 05 — Port the Hermes install to Linux

Goal: the Linux side runs Hermes Agent natively, off the same profile, with the same skills, plugins,
cron jobs and memory you had on Windows — minus the parts that are Windows-only.

The cleanest way to move the profile is a **payload tarball staged on the data volume** (built in
Windows, unpacked in Linux), because an encrypted C: is unreadable from Linux (see
[04-first-boot.md](04-first-boot.md)). Your Windows Hermes is still running; you are **copying**,
never moving. Nothing on the Windows side is deleted by this port.

## 5.1 What transfers, what re-paths, what you rebuild

| Profile part | Windows path (under the profile) | On Linux | Why |
| --- | --- | --- | --- |
| `skills/` | `skills/` | **Copies as-is** | Plain text; no OS dependency |
| `plugins/` | `plugins/` | **Copies** (check for `.exe`/native deps) | Mostly text; anything with a compiled binary needs a Linux build |
| `cron/` | `cron/` | **Copies**, then re-point paths | The job list is portable; job bodies that name `C:\…` paths need editing |
| Memory (`MEMORY.md`, `USER.md`) | `memories/` | **Copies**, then re-scrub | Persistent memory is text, but the *content* names Windows paths and the username |
| `config.yaml` | `config.yaml` | **Re-enter secrets, fix paths** | Provider keys live in `.env`; paths and shell differ |
| `.env` | `.env` | **Copies** | The live keys. Treat the whole payload as a secret (`chmod 600`) |
| Session history / `state.db` | `*.db` | **Copies** | SQLite; platform-independent. `state.db` can be > 100 MB |
| Installed MCP servers / native tools | varies | **Re-install** | Anything compiled for Windows must be rebuilt for Linux |
| Scheduled tasks (Windows Task Scheduler), app shortcuts | n/a | **Re-create** | OS-level, not part of the profile |

## 5.2 Install Hermes on Linux

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | sh
# then confirm the CLI is on PATH
hermes --version
```

The installer creates the Linux profile directory (`~/.local/share/hermes` or `~/.config/hermes`,
depending on build). Find it, because you will copy your Windows profile **on top of** it:

```bash
hermes --help | head -20
ls -d ~/.local/share/hermes ~/.config/hermes 2>/dev/null
```

## 5.3 Unpack the profile from the payload

```bash
cd ~/hermes-port
tar -xzf hermes-port-payload.tar.gz        # produces a profile/ tree
# Inspect before overwriting anything:
find profile -maxdepth 2 -type d | head -40
```

Read the tree. You should see `skills/`, `plugins/`, `cron/`, `memories/`, `config.yaml`, `.env`,
and the `*.db` files. If a directory is missing, the payload build skipped it — see
[payload-and-checksums.md](payload-and-checksums.md).

Back up the fresh Linux profile before merging, then copy your Windows profile over it:

```bash
PROFILE=~/.local/share/hermes          # <-- the value you printed in 5.2
cp -a "$PROFILE" "${PROFILE}.fresh-backup"
cp -a ~/hermes-port/profile/. "$PROFILE"/
```

## 5.4 Fix what does not survive a path change

Persistent memory and cron job bodies name Windows paths. Fix them, do not leave them stale:

```bash
# Find every Windows path that leaked into the profile
grep -rInF -e 'C:\' -e 'AppData' -e 'OneDrive' "$PROFILE"/memories "$PROFILE"/cron 2>/dev/null

# The mappings to apply (edit the files, do not blind-sed a whole tree):
#   C:\Users\<you>\...           -> /home/<you>/...
#   %LOCALAPPDATA%\hermes        -> the Linux profile dir from 5.2
#   OneDrive vault path          -> a Linux path you will recreate
```

Then re-scrub the copied memory: your Windows `MEMORY.md`/`USER.md` still say the machine has C:/D:
volumes, a OneDrive vault, and a Windows username. Update the facts that changed (paths, drives) and
leave the rest. A memory that points at `C:\…` on a machine with no C: is worse than no memory.

## 5.5 Skills, plugins, cron

```bash
# Skills are the whole point — confirm the count matches Windows
find "$PROFILE"/skills -name SKILL.md | wc -l
# Plugins: list them, and note any with a compiled/binary component
ls "$PROFILE"/plugins
# Cron: list the jobs, then fix any with Windows paths in the body
ls "$PROFILE"/cron
```

Re-install or rebuild:

- Any **plugin or MCP server** with a native/compiled component — rebuild it from source on Linux.
- Any **skill** that shells out to a Windows-only tool (`powershell.exe`, `.exe` paths). The two
  dual-boot skills on the reference machine needed no change; a skill that calls `powershell.exe`
  needs a rewritten command.

## 5.6 Secrets and the messaging gateway

If your Hermes talks to a chat platform, the gateway and its session are **per-platform**. Re-enter
the bot token on Linux and **run only one poller at a time** across the two OSes:

- A Telegram/Discord **long-poll** session held by Windows will 409/conflict if Linux also polls with
  the same token. Either stop the Windows poller before starting Linux, or use one bot token per side.
- Re-enter provider keys that did not come through `.env`, and confirm with a trivial prompt:

```bash
hermes -p "reply with the single word: ready"
```

## Checkpoint 5

- [ ] `hermes --version` runs on Linux.
- [ ] `find "$PROFILE"/skills -name SKILL.md | wc -l` equals the Windows skill count.
- [ ] `grep -rInF -e 'C:\' -e 'AppData' -e 'OneDrive' "$PROFILE"/memories` returns **nothing**.
- [ ] A test prompt returns a real model response (provider key works).
- [ ] No Windows-only poller is running against the same bot token.
- [ ] The Windows Hermes still works — you copied, you did not move.

Next: [06-robotics-stack.md](06-robotics-stack.md) (optional) or [07-verification.md](07-verification.md).
