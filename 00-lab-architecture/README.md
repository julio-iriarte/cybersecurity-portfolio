# 00 · Lab Architecture

## Goal
Isolated virtual lab to practice hardening, detection, and incident response safely, with a Wazuh SIEM as the central monitoring point.

## Host
| Item | Value |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2 (Extension Pack not installed — not required, smaller attack surface) |
| CPU / RAM | AMD Ryzen 7, 8 cores / 16 threads · ~28 GB RAM |
| Storage | Internal SSD for the SIEM · external SSD for Windows Server and ISO images |

## Inventory (sanitized)

| VM | OS | Role | IP | Status | Wazuh agent |
|---|---|---|---|---|---|
| wazuh-manager | Ubuntu Server 26.04.1 LTS | SIEM — Wazuh 4.14 all-in-one (manager, indexer, dashboard) | 10.10.10.10 | ✅ Deployed | — (server) |
| OPNsense | OPNsense (FreeBSD) | Lab firewall and gateway | 10.10.10.1 | 🟡 Pending configuration | — |
| DC01 | Windows Server 2022 | Domain Controller (AD DS, DNS) | 10.10.10.11 | ⚪ Planned | ⚪ Planned |
| W11Pro | Windows 11 Pro | Domain client / Windows endpoint | 10.10.10.12 | 🟡 Isolated, not yet configured | ⚪ Planned |
| LinuxMint | Linux Mint | Linux endpoint | 10.10.10.13 | 🟡 Isolated, not yet configured | ⚪ Planned |
| Kali | Kali Linux 2025.2 | Controlled test-event generation to validate detections | 10.10.10.50 | ✅ Isolated | — |
| MS | Metasploitable | Intentionally vulnerable target | 10.10.10.65 | ✅ Isolated | — |
| MSF2 | Metasploitable2 | Intentionally vulnerable target | 10.10.10.66 | ✅ Isolated | — |

## Network design

```
              Internet
                 │
          (planned) OPNsense  10.10.10.1
                 │  WAN: NAT · LAN: lab-net
                 │
   ── lab-net · VirtualBox Internal Network · 10.10.10.0/24 ──
     │          │        │         │          │        │
  wazuh-mgr   DC01    W11Pro   LinuxMint     Kali    MS / MSF2
    .10        .11      .12       .13         .50    .65 / .66
```

- **All lab VMs use a VirtualBox Internal Network (`lab-net`)**: no bridged or host-only adapters. VMs can reach each other, but not the home network or the physical host.
- **Promiscuous mode: Deny** on lab adapters (enabled only when a traffic-analysis exercise requires it).
- **Temporary Internet access:** `wazuh-manager` has a second adapter on VirtualBox NAT, used only for installation and updates. It will be removed once OPNsense becomes the lab gateway.
- **Administrative access:** SSH to `wazuh-manager` through a NAT port-forward bound to `127.0.0.1` only, so it is reachable from the physical host and not from the home network.
- **Vulnerable targets (MS, MSF2) will never get Internet egress** (to be enforced by OPNsense firewall rules).

## Baseline
- Snapshot `pre-wazuh` taken before installing the SIEM: ✅
- Snapshots `clean-baseline` for endpoints: ⬜ pending
- Wazuh SCA (CIS) initial score per host: ⬜ pending (after agent deployment)

## Related write-ups
- [Session 01 — Lab isolation and SIEM deployment](../writeups/session-01-lab-baseline.md)
