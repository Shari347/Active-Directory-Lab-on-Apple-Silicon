# Active-Directory-Lab-on-Apple-Silicon
Deploying Windows Server 2019 via x86_64 Emulation

## 💡 Background & Motivation
Most Active Directory home lab guides assume Intel/AMD hardware running VirtualBox. Windows Server has no native ARM64 build, so running this lab on an Apple Silicon MacBook meant none of the standard steps worked out of the box — from VM creation through to networking.

This project documents the full build **and** the architecture-specific problems that came up along the way: firmware mismatches between ARM64 and x86_64 virtual machines, OOBE setup failures under emulation, and a networking conflict between UTM's built-in DHCP and the domain controller's own DHCP service — none of which are covered in standard AD lab tutorials.

## 📂 Contents

- [Objective](#-objective)
- [Lab Architecture](#️-lab-architecture)
- [Prerequisites](#prerequisites)
- [Step 1: Create and Configure the Domain Controller VM](#step-1-create-and-configure-the-domain-controller-dc-vm)
- [Step 2: Booting Up the VM](#step-2-booting-up-the-vm)
- [Step 3: Configuring IP Addressing](#step-3-configuring-ip-addressing)
- [Step 4: Install AD DS and Create a Domain](#step-4-install-active-directory-domain-services-ad-ds-and-create-a-domain)
- [Step 5: Create a Domain Admin Account](#step-5-create-a-domain-admin-account)
- [Step 6: Install and Set Up RAS/NAT](#step-6-install-and-step-up-rasnat)
- [Step 7: Set Up DHCP](#step-7-setting-up-a-dhcp-server-on-our-domain-controller)
- [Step 8: Disable IE Enhanced Security Configuration](#step-8-disabling-ie-enhanced-security-configuration-to-browse-the-internet)
- [Step 9: Bulk User Provisioning with PowerShell](#step-9-bulk-user-provisioning-with-powershell)
- [Step 10: Create the Windows 10 Client and Join the Domain](#step-10-create-the-windows-10-client-client1-and-join-the-domain)
- [Notes & Known Issues](#notes--known-issues)
- [What You Can Do With This Lab](#what-you-can-do-with-this-lab)

## 🎯 Objective

Simulate a small enterprise Active Directory environment end-to-end, on hardware it wasn't designed to run on:

1. Deploy a Windows Server 2019 domain controller under x86_64 emulation on Apple Silicon
2. Install and configure Active Directory Domain Services (AD DS) and DNS
3. Configure NAT/RAS so the internal network can route out to the internet
4. Set up a DHCP server and scope for automatic client addressing
5. Bulk-provision 1,000+ user accounts with PowerShell
6. Join a Windows 10 client to the domain and validate authentication
7. Diagnose and document the emulation-specific issues that don't show up on standard hardware

## 🏗️ Lab Architecture

| Component | Role | Software |
|---|---|---|
| Host machine | Hypervisor | UTM (QEMU, x86_64 emulation) |
| VM 1 | Domain Controller | Windows Server 2019 + AD DS |
| VM 2 | Domain-joined client | Windows 10 |

```
[ Host: UTM (QEMU, x86_64 Emulation) ]
        |
        |-- VM 1: Windows Server 2019
        |       └── Domain Controller
        |               └── Active Directory Domain Services
        |               └── DNS
        |               └── DHCP
        |               └── RAS/NAT
        |
        └── VM 2: Windows 10 Client
                └── Joined to the AD domain
```

The lab domain used in this build is `mydomain.com`.

**Network layout:**

| NIC | Attached to | Role | IP Config |
|---|---|---|---|
| DC — NIC1 | Shared Network | Internet-facing | DHCP (from host) |
| DC — NIC2 | Host Only | Internal network | Static — `172.16.0.1`, mask `255.255.255.0` |
| Client1 — NIC | Host Only | Internal network | Static — `172.16.0.101`* |

\* *Intended to be DHCP-assigned from the DC's scope (`172.16.0.100–200`), but set statically due to a UTM networking conflict — see Notes & Known Issues.*

Built on Apple Silicon, this lab required **x86_64 emulation (UTM/QEMU)** in place of VirtualBox, since Windows Server has no native ARM64 build.

## Prerequisites

- **Windows Server 2019 ISO** — from Microsoft's "Get started for free" evaluation page, click **Download the ISO** (not "Try on Azure" or "Download the VHD")
- **Windows 10 ISO** — downloaded the same way from Microsoft
- **[UTM](https://mac.getutm.app)** — used in place of VirtualBox, since VirtualBox can't run x86_64 Windows Server on Apple Silicon

## Step 1: Create and Configure the Domain Controller (DC) VM

Before first boot, set up the VM with the right architecture and networking so you don't hit firmware mismatches later.

**Enable bi-directional clipboard sharing and drag-and-drop** (optional, but makes moving files in/out of the VM easier)

**Create the VM**
- Click **+** → choose **Emulate** (not Virtualize)
- Architecture: **x86_64** (scroll to find it — not the ARM64 default)
- System: leave default (Q35 chipset)

**Windows-specific screen**
- Leave "Install Windows 10 or higher" unchecked
- Browse and select `SERVER2019.iso`
- Check **UEFI Boot**, leave Secure Boot/TPM unchecked
- Continue

**Shared Directory screen**
- Leave blank, Continue

**RAM / Storage**
- RAM: at least 4096 MB
- Disk: 64GB+

**Networking (two adapters)**
- NIC1: Network Mode = **Shared Network** (internet-facing)
- NIC2: Network Mode = **Host Only** (internal network)
- Emulated Network Card: Intel Gigabit Ethernet (e1000) on both
























