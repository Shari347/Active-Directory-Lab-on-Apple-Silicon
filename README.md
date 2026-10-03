# Active-Directory-Lab-on-Apple-Silicon
Deploying Windows Server 2019 via x86_64 Emulation

## 💡 Background & Motivation
Most Active Directory home lab guides assume Intel/AMD hardware running VirtualBox. Windows Server has no native ARM64 build, so running this lab on an Apple Silicon MacBook meant none of the standard steps worked out of the box — from VM creation through to networking.

This project documents the full build **and** the architecture-specific problems that came up along the way: firmware mismatches between ARM64 and x86_64 virtual machines, OOBE setup failures under emulation, and a networking conflict between UTM's built-in DHCP and the domain controller's own DHCP service — none of which are covered in standard AD lab tutorials.

## 🎯 Objective

Simulate a small enterprise Active Directory environment end-to-end, on hardware it wasn't designed to run on:

1. Deploy a Windows Server 2019 domain controller under x86_64 emulation on Apple Silicon
2. Install and configure Active Directory Domain Services (AD DS) and DNS
3. Configure NAT/RAS so the internal network can route out to the internet
4. Set up a DHCP server and scope for automatic client addressing
5. Bulk-provision 1,000+ user accounts with PowerShell
6. Join a Windows 10 client to the domain and validate authentication
7. Diagnose and document the emulation-specific issues that don't show up on standard hardware

