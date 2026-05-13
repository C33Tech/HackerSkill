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
