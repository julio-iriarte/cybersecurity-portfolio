# 00 · Lab Architecture

## Goal
Isolated virtual lab to practice hardening, detection, and incident response safely.

## Inventory (sanitized)

| Host | OS | Role | IP | Wazuh agent |
|---|---|---|---|---|
| WAZUH01 | Windows Server | SIEM (Wazuh manager) | 10.10.10.10 | — |
| DC01 | Windows Server | Domain Controller | 10.10.10.11 | Vms |
| WIN-CLIENT01 | Windows 11 | Windows client | 10.10.10.12 | Vms |
| LNX-SRV01 | Ubuntu Server 26.04 LTS | Linux server | 10.10.10.13 | Vms |

## Network design
- VirtualBox network type: `Host-only`
- Isolation from home/corporate network: `<describe>`
- Diagram: `diagrams/lab-diagram.png`

## Baseline
- "Clean baseline" snapshots taken: ⬜
- Wazuh SCA (CIS) initial score per host: _pending_
