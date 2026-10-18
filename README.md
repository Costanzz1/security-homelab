# Segmented Security Home Lab (Proxmox + pfSense + Wazuh)

A self-built, isolated cybersecurity lab for practicing attack and detection. Vulnerable targets live on an isolated virtual network behind a pfSense firewall, monitored by a Wazuh SIEM.

> **Status:** 🚧 In progress

---

## 🗺️ Architecture

```
Home router (192.168.x.x)
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
| pfSense | Firewall / router / DHCP | vmbr0 + vmbr1 | 2 | 2 GB | 16 GB |
| Kali | Attacker + admin workstation | vmbr1 | 2 | 4 GB | __ GB |
| Wazuh | SIEM | vmbr1 | 4 | __ GB | 50 GB |
| Metasploitable 2 | Vulnerable target | vmbr1 | 1 | 1 GB | __ GB |
| Windows | Monitored endpoint | vmbr1 | 2 | 4 GB | __ GB |

---

## 🔨 Build Log

### Step 1: Proxmox prep
Disabled the enterprise repositories (they require a paid subscription) and enabled the free no-subscription repository so the host can receive updates.

![Proxmox repositories](screenshots/01-proxmox-repos.png)

**Notes / what broke:**
Lost access to the Proxmox web UI. My laptop was on a different subnet (192.168.0.x vs 192.168.100.x). On the host, `ip a` showed the physical link down; the Ethernet cable had come unseated. After reseating, `vmbr0` was UP but had no IP until I reapplied the config with `ifreload -a`.
**Lesson:** check the physical layer first, then confirm the interface actually has its address.

### Step 2: Lab network (vmbr1)
- [ ] 📸 System → Network page showing vmbr0 and vmbr1

**Notes / what broke:**


### Step 3–5: pfSense
- [ ] 📸 pfSense VM Hardware tab (two NICs)
- [ ] 📸 Console showing WAN/LAN interface assignment
- [ ] 📸 pfSense dashboard after setup wizard

**Notes / what broke:**


### Step 6: Kali
- [ ] 📸 Kali VM Hardware tab
- [ ] 📸 Kali receiving a 10.10.10.x address (`ip a`)

**Notes / what broke:**


### Step 7: Firewall rules
- [ ] 📸 LAN rules page
- [ ] 📸 Test: `ping 8.8.8.8` succeeds, ping to home router fails

**Why each rule exists:**

| Rule | Action | Source | Destination | Reason |
|---|---|---|---|---|
| 1 | Block | LAN net | Home subnet | Prevent lab → home network access |
| 2 | Allow | LAN net | Any | Internet access for updates |

**Notes / what broke:**


### Step 8: Wazuh
*(coming next)*

---

## 📚 What I Learned

-

## ⏭️ Next

- [ ] Wazuh SIEM deployment
- [ ] Windows endpoint + agents
- [ ] Attack → detect write-ups
