# [Session 01] — Lab Isolation Review and SIEM Deployment

**Date:** 2026-10-27 · **Block:** 00 / 01 · **Time spent:** several 2 h sessions
**Frameworks:** CIS Controls v8 (4 – Secure Configuration, 12 – Network Infrastructure Management) · NIST CSF 2.0 (PR.PS, ID.RA)

## 🎯 Objective
Review the security of my virtual lab before using it, isolate every VM from my home network, and deploy a Wazuh SIEM as the central monitoring point.

## 🧠 Context
A security lab contains intentionally vulnerable machines and attack tools. If it is not isolated, the lab itself becomes a risk to the home network. Before practicing detection, the environment must be safe, documented, and recoverable.

## 🔍 Findings

| # | Finding | Risk | Status |
|---|---|---|---|
| F1 | Metasploitable2 (intentionally vulnerable VM) was attached to a **bridged** adapter | 🔴 Critical | ✅ Fixed |
| F2 | Lab VMs were spread across two hosts with mixed network modes (bridged, NAT, internal) | 🟠 High | ✅ Fixed |
| F3 | After a VirtualBox upgrade, the Extension Pack stayed on the old version and VMs failed to start (`VERR_PDM_DRIVER_NAME_CLASH`) | 🟡 Medium | ✅ Fixed |
| F4 | The SIEM runs on Ubuntu 26.04, which is not in Wazuh 4.14's supported OS list | 🟡 Medium | ⚠️ Risk accepted |
| F5 | The Ubuntu installer allocated only ~50 % of the disk to the root LVM volume | 🟢 Low | ✅ Fixed |

### F1 — Vulnerable VM exposed to the home network
- **Risk:** any compromised device on my home network could reach a host with known backdoors and default credentials, and use it to pivot.
- **Remediation:** moved the VM to an isolated VirtualBox Internal Network (`lab-net`), promiscuous mode set to Deny.
- **Verification:**
```text
VBoxManage showvminfo "<vm>" --machinereadable
nic1="intnet"
intnet1="lab-net"
```

### F2 — Inconsistent network modes
- **Remediation:** every lab VM moved to `lab-net`. A VM used for non-lab purposes stays outside the lab and is never connected to `lab-net`, so it cannot become a bridge between networks.
- **Decision:** consolidate the whole lab on one host. VirtualBox internal networks only exist inside a single host, so VMs on different hosts cannot talk to each other.

### F3 — Hypervisor components out of sync after an upgrade
- **Root cause:** VirtualBox was upgraded but the Extension Pack was not, so two versions of the same modules loaded at VM start.
- **Remediation:** removed the outdated Extension Pack (not required by this lab — fewer components, smaller attack surface) and restarted the host.
- **Lesson:** when upgrading a hypervisor, every component (extension packs, guest additions, drivers) must move to the same version. This belongs in the change-management checklist.

### F4 — SIEM on an unsupported OS (risk accepted)
- **Context:** Wazuh 4.14 lists Ubuntu 16.04–24.04 as supported. I chose Ubuntu 26.04, which I already run successfully on another server.
- **Compensating controls:**
  - Snapshot `pre-wazuh` taken before installation as a rollback plan.
  - Wazuh package updates frozen, to avoid partial upgrades of the three components.
  - Installer run with `-i` (ignore OS check) as a documented, intentional decision.
- **Review:** revisit when Wazuh officially supports Ubuntu 26.04.

### F5 — Root volume using half of the disk
```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
# Result: / grew from 24 GB to 48 GB
```

## 🛠️ SIEM deployment
1. Downloaded the Ubuntu Server ISO and **verified its SHA256 hash** against the official `SHA256SUMS` file (supply-chain control).
2. Created the VM from the command line (4 vCPU, 8 GB RAM, 50 GB dynamic disk, UTC clock, audio and USB disabled):
```powershell
VBoxManage createvm --name "Wazuh" --ostype Ubuntu_64 --register
VBoxManage modifyvm "Wazuh" --cpus 4 --memory 8192 --rtc-use-utc on --audio-enabled off --usb-ohci off
VBoxManage modifyvm "Wazuh" --nic1 nat --nic2 intnet --intnet2 "lab-net" --nicpromisc2 deny
```
3. Configured a static IP on the lab network with a dedicated netplan file (`10.10.10.10/24`, no gateway on this interface to avoid routing conflicts with the temporary NAT adapter).
4. Enabled SSH administration through a NAT port-forward **bound to `127.0.0.1`**, so the SIEM is not reachable from the home network.
5. Downloaded the Wazuh installation script first (instead of piping `curl` into `bash`), then installed Wazuh 4.14 all-in-one.

## ✅ Result
- All lab VMs isolated on `lab-net` (10.10.10.0/24).
- Wazuh SIEM deployed at `10.10.10.10`.
- Lab inventory and network design documented in [00-lab-architecture](../00-lab-architecture/).

## 🛡️ Prevention · Detection · Containment
| Prevention | Detection | Containment |
|---|---|---|
| Network isolation of the lab, ISO hash verification, minimal VM hardware | Wazuh SIEM deployed as central log and alert platform | Snapshots as rollback points; vulnerable targets kept without Internet egress |

## 📚 Lessons learned
- **Verify with evidence, not memory:** one VM I believed was isolated was still on a bridged adapter.
- **Default settings are risky:** VirtualBox port-forwarding listens on all interfaces unless you set the host IP to `127.0.0.1`.
- **Check the vendor support matrix before deploying critical tools.** When you choose to deviate, document it as an accepted risk with compensating controls.
- **Always have a rollback plan** before a major change.

## ⏭️ Next steps
- Access the Wazuh dashboard and take a `post-wazuh-ok` snapshot.
- Configure OPNsense as the lab gateway and remove the temporary NAT adapter.
- Deploy DC01 (Windows Server 2022) and the Wazuh agents, then record the CIS (SCA) baseline.

## 🔗 References
- Wazuh Quickstart and requirements — https://documentation.wazuh.com/current/quickstart.html
- VirtualBox networking modes — https://www.virtualbox.org/manual/
- CIS Controls v8 — https://www.cisecurity.org/controls
