# Pwn Challenge Skill — Design

A Claude Code skill that acts as a full-lifecycle coach for offensive security
challenges: Hack The Box, TryHackMe, CTFs, and any authorized black-box target.

## Goals

- Coach the user through recon → foothold → privilege escalation → root.
- Maintain durable per-target state in a markdown notes file the user can read.
- Preserve learning: give tiered hints by default, escalate on request.
- Execute safe enumeration automatically; ask before exploitation or auth attempts.
- Work on any target (HTB seasonal, THM, CTFs, custom labs) — not just well-known boxes.

## Non-Goals

- Operating against unauthorized targets.
- Fully autonomous exploitation (the skill always confirms before running exploits).
- Replacing the user's own enumeration; the skill assumes intermediate proficiency.

## Audience & Tone

- Intermediate operator. Skill assumes familiarity with nmap, burp, common shells.
- Does not over-explain basics. Does explain *why* a suggested step matters when nudging.

## Architecture

A single skill `pwn-challenge` installed at `~/.claude/skills/pwn-challenge/`:

```
~/.claude/skills/pwn-challenge/
├── SKILL.md                       # main instructions Claude follows
├── references/
│   ├── playbook.md                # service & context technique reference
│   └── target-template.md         # markdown template for a new target
~/.claude/commands/
└── pwn.md                         # /pwn slash command
```

State for each engagement lives in the user's current working directory:

```
targets/
├── .current                       # name of active target (single line)
└── <target>/
    ├── notes.md                   # primary state file
    ├── scans/                     # raw tool output (nmap, gobuster, etc.)
    └── loot/                      # creds, hashes, downloaded binaries
```

`targets/.current` is a cheap pointer so cross-turn continuity does not require
guessing which box is active.

## Lifecycle Phases

The skill tracks one of four phases per target, recorded in `notes.md` as `Status:`:

1. **recon** — port/service discovery, web/SMB/etc. enumeration.
2. **foothold** — turning a finding into initial shell or authenticated access.
3. **priv-esc** — local enum, user → root (or lateral movement on AD).
4. **rooted** — both flags captured, writeup mode.

Each phase has an internal checklist the skill walks through against the current
notes.md, so it does not re-suggest work that has already been done.

## Notes File Template

`references/target-template.md`:

```markdown
# <Target> (<IP>)

**Status:** recon | foothold | priv-esc | rooted
**Platform:** HTB / THM / CTF / other
**Difficulty:** easy/med/hard  **OS:** linux/windows/unknown
**Started:** YYYY-MM-DD

## Targets
- IP:
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
- Evidence: scans/<file>, loot/<file>
- Commands that worked:

### Lateral / privilege escalation
### Root

## Flags
- user.txt / user flag:
- root.txt / root flag:

## Hints given
- Tier 1: …
- Tier 2: …
- Tier 3: …

## Things tried (didn't work)
-

## Open questions / next steps
-
```

Raw scan output is written to `targets/<target>/scans/<tool>-<timestamp>.txt`
(via `-oN` or `tee`), not pasted into `notes.md`. A one-line summary is appended
to the relevant `notes.md` section after each scan.

The skill **reads the entire `notes.md`** at the start of every turn so it has
full state. It appends or edits sections rather than silently rewriting prior
content.

## Tiered Hint Protocol

When the user asks for a hint, asks "what now?", or says they're stuck, the
skill responds at **Tier 1** by default and escalates only on request.

- **Tier 1 — Nudge.** Point at an area without naming the technique.
  Example: *"You enumerated SMB but didn't check for null sessions on shares —
  worth a look."*
- **Tier 2 — Technique.** Name the tool or class of attack, not the exact payload.
  Example: *"That CMS version has a known LFI. Look at how the `file=` param is
  handled."*
- **Tier 3 — Solution.** Provide the command, payload, or exploit directly.

The user escalates with phrases like *"more"*, *"next hint"*, *"give it to me"*.

**Override phrases:**
- *"spoil it"* / *"solve mode"* — jump to Tier 3 for the current question only.
- *"no spoilers"* — lock to Tier 1 until released with *"hints ok"*.

Each hint given is logged under **Hints given** in `notes.md` so the skill does
not repeat itself across turns.

## Execution Rules

Every proposed command falls into one of three buckets.

### Auto-run (no prompt)

- Port/service scans: `nmap` (non-intrusive), `rustscan`.
- Web fingerprinting: `whatweb`, `curl -I`, `wafw00f`.
- Directory/vhost enumeration: `gobuster`, `ffuf`, `feroxbuster`.
- DNS/SMB info: `dig`, `smbclient -L`, `enum4linux -A`, `rpcclient` info queries.
- Read-only commands on shells the user has already established.

These run via Bash. Output is teed to `targets/<target>/scans/` and summarized
into `notes.md`.

### Confirm first (skill proposes, waits for explicit "yes")

- Any authentication attempt: `hydra`, `crackmapexec` with credentials, ssh login.
- Exploit execution: Metasploit modules, public PoCs, custom payloads.
- Anything spawning a reverse or bind shell.
- Password cracking with significant CPU cost (hashcat/john with large wordlists).
- Writes or uploads to the target.

The skill prefaces each confirm-required command with a one-line *why*.

### Never auto

- Destructive actions on the target (rm, service stop, format).
- Anything touching hosts outside the assigned target IP.
- Bruteforce against rate-limited or lockout-prone services without explicit
  user authorization for that specific service.

## Playbook & Web Lookups

`references/playbook.md` is a curated, dense reference loaded **on demand**, not
at skill activation. The skill loads only the section relevant to current
findings (e.g., port 445 open → load the SMB section).

**Organization:**

- Per-service: SMB, FTP, SSH, HTTP/HTTPS, RDP, WinRM, LDAP, Kerberos, MSSQL,
  MySQL, Redis, SNMP, NFS.
- Per-context: Linux priv-esc checklist, Windows priv-esc checklist, AD attack
  paths (kerberoasting, AS-REP, DCSync, etc.), common web vulnerability classes
  (LFI→RCE, SSTI, SSRF, deserialization).

**Per-entry format:** what to enumerate → common tools/commands → common
findings → next-step pointers.

**Web lookups** (via `WebSearch` / `WebFetch`) are used for:

- CVE details when `notes.md` records a specific version.
- Exploit-DB or GitHub PoCs by CVE.
- HackTricks, GTFOBins, LOLBAS pages for specific techniques.
- Current syntax for tools when the built-in playbook is vague.

Preferred sources: HackTricks and GTFOBins first; general search as fallback.

## Slash Command `/pwn`

Defined at `~/.claude/commands/pwn.md`.

- `/pwn <target> <ip>` — start a new target. Creates `targets/<target>/` with
  `notes.md` from template, fills in the IP, and kicks off the initial nmap
  scan automatically.
- `/pwn <target>` — resume. Reads `targets/<target>/notes.md`, summarizes
  current state, asks where to focus.
- `/pwn` (no args) — list targets under `./targets/` with their phase status,
  ask which to resume.
- `/pwn done` — mark the active target as `rooted`, offer to generate a writeup
  from `notes.md`.

## Auto-Activation

The skill's `description` field includes keywords that cause the harness to
load it automatically:

- Platform terms: `hackthebox`, `htb`, `tryhackme`, `thm`, `ctf`.
- Target subnets: `10.10.10.`, `10.10.11.`, `10.129.` (HTB seasonal).
- Generic markers: `user.txt`, `root.txt`, `flag.txt`, "got a shell", "rooted",
  pasted nmap output (multiple lines matching `^\d+/(tcp|udp)\s+open`).

When auto-activated and a `targets/` directory exists in the cwd, the skill
identifies the active target by:

1. Target named explicitly in the message, else
2. Contents of `targets/.current`, else
3. Most recently modified `targets/*/notes.md`, else
4. Ask the user.

## Authorization Guardrail

The first line of `SKILL.md` instructs the model to refuse to engage if the
target is not an authorized lab/CTF/engagement. The user states the platform
(HTB, THM, CTF, etc.) at `/pwn` start; that constitutes the authorization
context. If the user attempts to point the skill at a real-world IP outside
known lab ranges and does not state explicit written authorization, the skill
asks before proceeding.

## Open Questions

None at design time.
