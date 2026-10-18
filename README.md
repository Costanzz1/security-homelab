# Segmented Security Home Lab (Proxmox + pfSense + Wazuh)

A self-built, isolated cybersecurity lab for practicing attack and detection. Vulnerable targets live on an isolated virtual network behind a pfSense firewall, monitored by a Wazuh SIEM.

> **Status:** 🚧 In progress

---

## 🗺️ Architecture

```
Home network (192.168.0.0/16)
      │
  Host NIC ── vmbr0 (home network)
                 │
            [pfSense VM]  WAN = vmbr0 | LAN = vmbr1
                 │
              vmbr1 (LAB · 10.10.10.0/24 · no physical port)
     ┌───────────┼────────────┬───────────────┐
   Kali        Wazuh     Metasploitable    Windows
```

**Key design decision:** `vmbr1` has no physical network port, so all lab traffic is forced through pfSense. A firewall rule blocks the lab from reaching the home network, so a compromised VM can't touch real devices.

## 🖥️ Hardware

| Component | Spec |
|---|---|
| Host | HP ProDesk 600 G2 SFF |
| CPU | Intel i5-6500 (4 cores) |
| RAM | 16 GB DDR4 |
| Storage | 128 GB SSD |
| Hypervisor | Proxmox VE 9.2 |

## 🧱 Virtual Machines

| VM | Role | Network | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| pfSense | Firewall / router / DHCP | vmbr0 + vmbr1 | 1 | 2 GB | 16 GB |
| Kali | Attacker + admin workstation | vmbr1 | 2 | 4 GB | 32 GB |
| Wazuh | SIEM | vmbr1 | 4 | __ GB | __ GB |
| Metasploitable 2 | Vulnerable target | vmbr1 | 1 | 1 GB | __ GB |
| Windows | Monitored endpoint | vmbr1 | 2 | 4 GB | __ GB |

---

## 🔨 Build Log

### Step 1: Proxmox prep
Disabled the enterprise repositories (they require a paid subscription) and enabled the free no-subscription repository so the host can receive updates.

**Notes / what broke:**
Lost access to the Proxmox web UI. My laptop was on a different subnet (192.168.0.x vs 192.168.100.x). On the host, `ip a` showed the physical link down; the Ethernet cable had come unseated. After reseating, `vmbr0` was UP but had no IP until I reapplied the config with `ifreload -a`.
**Lesson:** check the physical layer first, then confirm the interface actually has its address.

### Step 2: Lab network (vmbr1)
Created an isolated Linux bridge (`vmbr1`) with **no IP address and no physical port**. Lab VMs connect only to this bridge, so their only path in or out is through the pfSense firewall. Proxmox itself has no address on the lab network either.

<!-- ![Proxmox network bridges](screenshots/02-vmbr1-network.png) -->

### Step 3: pfSense installation
Installed **pfSense CE 2.9.0** using the Netgate online installer. The VM has two VirtIO network cards:

| Proxmox NIC | Bridge | pfSense interface | Role | Address |
|---|---|---|---|---|
| net0 | vmbr0 | vtnet0 | WAN | DHCP from home router |
| net1 | vmbr1 | vtnet1 | LAN | 10.10.10.1/24 (static) |

- **LAN DHCP pool:** 10.10.10.100–200, leaving .2–.99 free for static addresses (e.g., the SIEM)
- **Disk:** ZFS stripe on a single virtual disk; redundancy is handled by Proxmox snapshots instead
- **Edition:** pfSense CE (free) rather than pfSense Plus

**Notes / what broke:**
- The Netgate installer is an online installer, so the WAN interface needed working internet access *during* installation.
- The VM rebooted before I could detach the ISO. Proxmox's boot order tried the disk first, so it booted into the installed system anyway; I detached the ISO afterwards to prevent booting back into the installer.

### Step 4: Setup wizard decisions
- **Unchecked "Block RFC1918 private networks" on WAN.** My WAN sits on my home network (192.168.x.x), which is itself private address space. Leaving this on would block legitimate access from my home network later (e.g., the SIEM dashboard). This does *not* open anything: pfSense still blocks all unsolicited inbound WAN traffic by default.
- **Kept "Block bogon networks" on**, since reserved/unassigned address ranges should never appear as legitimate sources.
- **Configured NTP and the correct timezone.** Accurate time is critical for correlating firewall logs with SIEM alerts during an investigation.
- **Changed the default admin password.**

**Discovery:** pfSense's WAN received a `192.168.0.x` address, while the Proxmox host uses `192.168.100.x`. My home network has two subnets in use, which also explained earlier trouble reaching Proxmox from my laptop. This shaped the firewall rule below.

### Step 5: Kali Linux
Installed Kali 2026.2 on `vmbr1` only. It received `10.10.10.1xx` from pfSense's DHCP server and reached the internet through pfSense, confirming the full path: **Kali → pfSense LAN → pfSense WAN → home router → internet.**

Kali doubles as the admin workstation, since pfSense's web interface is only reachable from inside the lab.

**Notes / what broke:**
- The Kali installer stalled on "no sound card detected": I had selected the speech-synthesis install option. Restarted and chose Graphical Install.

### Step 6: Firewall rules (segmentation)
pfSense evaluates rules **top to bottom, first match wins**, so the block rule sits above the default allow rule.

| # | Action | Source | Destination | Reason |
|---|---|---|---|---|
| 1 | Block (logged) | LAN subnets | 192.168.0.0/16 | Prevent the lab from reaching any home network device, including the Proxmox host |
| 2 | Allow | LAN subnets | Any | Internet access for updates and tools |
| 3 | Disabled | LAN subnets | Any (IPv6) | The lab doesn't use IPv6, so it isn't allowed |

Using `/16` covers both home subnets (192.168.0.x and 192.168.100.x) with a single rule.

<!-- ![LAN rules](screenshots/05-lan-rules.png) -->

### Step 7: Verification
Tested from Kali:

| Test | Expected | Result |
|---|---|---|
| `ping 8.8.8.8` | Reachable (internet) | ✅ 0% loss |
| `ping 192.168.0.1` | Blocked (home router) | ✅ 100% loss |
| `ping 192.168.100.2` | Blocked (Proxmox host) | ✅ 100% loss |

Blocked attempts appear in pfSense's firewall log, which will later be forwarded to the SIEM.

<!-- ![Segmentation test](screenshots/06-segmentation-test.png) -->

Took `clean_base` snapshots of both VMs (powered off for consistency) before moving on.

### Step 8: Wazuh SIEM
*(coming next)*

---

## 📚 What I Learned

-

## ⏭️ Next

- [ ] Wazuh SIEM deployment
- [ ] Linux and Windows endpoints with agents
- [ ] Attack → detect write-ups
