# [Session 01] — Lab Isolation Review and SIEM Deployment

**Date:** 2026-10-27 (updated 2026-10-31) · **Block:** 00 / 01 · **Time spent:** several 2 h sessions
**Frameworks:** CIS Controls v8 (4 – Secure Configuration, 5 – Account Management, 12 – Network Infrastructure Management) · NIST CSF 2.0 (PR.PS, PR.AA, ID.RA, RS.AN)

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
| F6 | The dashboard `admin` password was unknown: the installer's credentials archive was missing | 🟡 Medium | ✅ Fixed |
| F7 | `wazuh-execd` failed and the dashboard could not obtain an API token (20 s timeout) | 🟠 High | ✅ Fixed (root cause in F8) |
| F8 | **Root cause:** Windows virtualization-based security (VBS) on the host forced VirtualBox into its slow NEM fallback, causing **kernel soft lockups** in the SIEM VM | 🔴 Critical | ✅ Fixed · ⚠️ Risk accepted |
| F9 | SIEM SSH administration needed from a second workstation without exposing it to the home network | 🟠 High | ✅ Hardened |

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

### F6 — Unknown dashboard admin password
- **Context:** the all-in-one installer stores generated credentials in `wazuh-install-files.tar`, but the file was not on the system.
- **Remediation:** reset only the `admin` user with the official tool, letting it **generate** the password so it never appears on the command line or in the shell history:
```bash
sudo bash -c 'cd /usr/share/wazuh-indexer/plugins/opensearch-security/tools && bash wazuh-passwords-tool.sh -u admin'
```
- **Residual risk:** the other internal users still have unknown passwords. **Planned:** rotate all of them (`-a`) and store them in a password manager.

### F7 — `wazuh-execd` failure and dashboard token timeout
- **Symptoms:** login worked but the dashboard showed *"Error getting the authorization token"* (timeout 20000 ms). The API log returned HTTP 500 errors.
- **Evidence:**
```text
/var/ossec/bin/wazuh-control status
wazuh-execd not running...      # stale PID: "Process ... not used by Wazuh"
```
- **Immediate fix:** `sudo systemctl restart wazuh-manager`, then a full reboot test.
- **Root cause:** not a Wazuh bug. The VM was being starved of CPU time by the host hypervisor configuration (see F8).

### F8 — Host VBS forced VirtualBox into NEM mode (root cause)
- **Symptoms:** after running overnight, the SIEM console showed kernel soft lockups (CPUs stuck for up to ~5 minutes), RCU stalls and systemd watchdog timeouts.
- **Evidence (VirtualBox VM log):**
```text
HM: HMR3Init: Attempting fall back to NEM: AMD-V is not available
```
  VirtualBox could not use the CPU's hardware virtualization directly, because the Windows hypervisor already owned it. It fell back to running on top of the Windows Hypervisor Platform (NEM), which is much slower and caused the lockups.
- **Investigation — each step was verified with evidence:**

  | Check | Result |
  |---|---|
  | Hyper-V role | Disabled — not enough |
  | `bcdedit hypervisorlaunchtype` | `Off` — not enough |
  | Memory integrity (HVCI) | Off — not enough |
  | `Win32_DeviceGuard.VirtualizationBasedSecurityStatus` | **`2` (running)** with **no security services configured** |
  | `HKLM\...\Control\DeviceGuard\EnableVirtualizationBasedSecurity` | **`1`** (Windows 11 default) — **the actual trigger** |

- **Remediation (with rollback plan):**
```powershell
reg export "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" DeviceGuard-backup.reg /y
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" EnableVirtualizationBasedSecurity 0 -Type DWord
bcdedit /set "{current}" vsmlaunchtype off
# reboot → VBS status 0 → VM log: "HM: HMR3Init: AMD-V w/ nested paging"
```
- **Side effect caught by change management:** a container workload on the same host (Docker Desktop on WSL2) **depends on the Windows hypervisor** and stopped working after the change. I rolled back, backed up and verified its data (SHA-256), migrated it to Docker Engine inside a Linux VM, and only then re-applied the change.
- **Risk accepted:** disabling VBS removes a kernel-isolation layer on the host. The practical loss is low, because **no VBS security service was actually running** (`SecurityServicesConfigured = {0}`).
  - **Compensating controls:** Microsoft Defender active, OS fully patched, the host is not used for general browsing, and vulnerable lab VMs have no Internet egress.
  - **Review:** if the host is ever used for sensitive work, re-enable VBS and move the lab to a dedicated hypervisor.
- **Related:** a *live* snapshot (which saves RAM) hung at 90 % while VirtualBox was in NEM mode. Since then, snapshots are taken **offline** (VM powered off).

### F9 — Hardened SSH administration of the SIEM
- **Starting point:** SSH was reachable only through a NAT port-forward bound to `127.0.0.1` on the host. This is secure, but there was no access from a second admin workstation.
- **Misconception avoided:** in VirtualBox NAT port-forwarding, the *Host IP* field is the **address the host listens on**, not a filter of allowed clients. Putting the workstation's IP there would simply fail.
- **Design (defense in depth):**

  | Layer | Control |
  |---|---|
  | Network | Second port-forward bound to `<host-lan-ip>:2222`. Windows Firewall inbound rule: TCP 2222, **remote address = `<admin-workstation-ip>` only**, **Private profile only**. |
  | Host profile | Home network switched from *Public* to *Private*, with network discovery and file sharing **turned off**. Existing firewall rules audited first. |
  | Authentication | **One ed25519 key per device**, each with a passphrase. Keys are separate from any non-lab credentials, and `IdentitiesOnly yes` is set per host alias. |
  | SSH daemon | `/etc/ssh/sshd_config.d/10-hardening.conf` (loads before vendor files, so its values win) |

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
- **Verification:**
```text
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin) '
passwordauthentication no
permitrootlogin no

# From the workstation, forcing password auth:
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <alias>
<user>@<host>: Permission denied (publickey).
```
- **Why one key per device:** a lost laptop means revoking one key, not all of them. The key fingerprint in `/var/log/auth.log` also tells **which device** logged in, which makes access attributable during an investigation.

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
- Wazuh SIEM deployed at `10.10.10.10`, stable after the host fix (no soft lockups, all manager daemons running).
- SIEM administration restricted to two authorized workstations, key-only.
- Lab inventory and network design documented in [00-lab-architecture](../00-lab-architecture/).

## 🛡️ Prevention · Detection · Containment
| Prevention | Detection | Containment |
|---|---|---|
| Network isolation of the lab, ISO hash verification, minimal VM hardware, key-only SSH with per-device keys, firewall scoped to one admin IP | Wazuh SIEM deployed as central log and alert platform; kernel logs checked for lockups after every host change | Offline snapshots as rollback points; registry and config backups before every change; vulnerable targets kept without Internet egress |

## 📚 Lessons learned
- **Verify with evidence, not memory:** one VM I believed was isolated was still on a bridged adapter.
- **Default settings are risky:** VirtualBox port-forwarding listens on all interfaces unless you set the host IP to `127.0.0.1`.
- **Check the vendor support matrix before deploying critical tools.** When you choose to deviate, document it as an accepted risk with compensating controls.
- **Always have a rollback plan** before a major change.
- **Symptoms are not causes:** the failing Wazuh service (F7) was a symptom. The real cause was two layers below, in the host's hypervisor configuration (F8).
- **"Disabled" is not "off":** removing Hyper-V and HVCI was not enough. Only the VBS status from `Win32_DeviceGuard` told the truth.
- **Map dependencies before a host-level change:** turning off the Windows hypervisor silently broke another workload that relied on it.
- **Understand what a setting really does:** the port-forward *Host IP* is a bind address, not an access control. Filtering belongs in the firewall.

## ⏭️ Next steps
- Confirm overnight stability of the SIEM and keep the `post-wazuh-ok` offline snapshot as the new baseline.
- Rotate all internal Wazuh passwords (F6 residual risk).
- Deploy the first Wazuh agent on the Linux endpoint (LinuxMint), then configure OPNsense as the lab gateway and remove the temporary NAT adapter.
- Deploy DC01 (Windows Server 2022) and record the CIS (SCA) baseline.

## 🔗 References
- Wazuh Quickstart and requirements — https://documentation.wazuh.com/current/quickstart.html
- Wazuh password management tool — https://documentation.wazuh.com/current/user-manual/user-administration/password-management.html
- VirtualBox networking modes — https://www.virtualbox.org/manual/
- Microsoft — Virtualization-based security (VBS) — https://learn.microsoft.com/windows-hardware/design/device-experiences/oem-vbs
- OpenSSH `sshd_config` manual — https://man.openbsd.org/sshd_config
- CIS Controls v8 — https://www.cisecurity.org/controls
