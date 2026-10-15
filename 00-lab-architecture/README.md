# 00 · Lab Architecture

## Goal
Isolated virtual lab to practice hardening, detection, and incident response safely.

## Inventory (sanitized)

| Host | OS | Role | IP | Wazuh agent |
|---|---|---|---|---|
| WAZUH01 | | SIEM (Wazuh manager) | 10.10.10.x | — |
| DC01 | | Domain Controller | 10.10.10.x | ⬜ |
| WIN-CLIENT01 | | Windows client | 10.10.10.x | ⬜ |
| LNX-SRV01 | | Linux server | 10.10.10.x | ⬜ |

## Network design
- VirtualBox network type: `<Internal Network / Host-only / NAT Network>`
- Isolation from home/corporate network: `<describe>`
- Diagram: `diagrams/lab-diagram.png`

## Baseline
- "Clean baseline" snapshots taken: ⬜
- Wazuh SCA (CIS) initial score per host: _pending_
