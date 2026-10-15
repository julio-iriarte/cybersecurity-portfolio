# 🧼 Sanitization checklist (before every commit)

Publishing lab work is good. Publishing sensitive data is a security incident.
Check **every item** before `git commit`.

- [ ] No real public IPs, domains, or hostnames from any employer or client
- [ ] Lab IPs and hostnames are generic (see table below)
- [ ] No usernames, emails, or personal data (except my public profile)
- [ ] No passwords, hashes, API keys, tokens, or Wazuh agent keys (`client.keys`)
- [ ] Screenshots reviewed: no browser tabs, taskbar, or notifications with private info
- [ ] Logs: only relevant excerpts, not full raw exports
- [ ] No malware samples or binaries
- [ ] Nothing from an employer's or client's real environment

## Naming conventions for the lab

| Real role | Public name |
|---|---|
| Domain Controller | `DC01.lab.local` |
| Windows client | `WIN-CLIENT01` |
| Linux server | `LNX-SRV01` |
| Wazuh manager | `WAZUH01` |
| Lab network | `10.10.10.0/24` |
