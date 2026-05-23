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
/plugin marketplace add C33Tech/HackerSkill
/plugin install pwn-challenge@C33Tech-HackerSkill
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
- `metasploit-framework` (`msfconsole`, `msfvenom`) — for guided exploitation, aux scanners, handlers, and post-exploitation

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
/plugin marketplace update HackerSkill
/plugin uninstall pwn-challenge@HackerSkill
```

## Source

[github.com/C33Tech/HackerSkill](https://github.com/C33Tech/HackerSkill)
