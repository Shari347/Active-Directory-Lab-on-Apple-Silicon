# Active-Directory-Lab-on-Apple-Silicon
Deploying Windows Server 2019 via x86_64 Emulation

## 💡 Background & Motivation
Most Active Directory home lab guides assume Intel/AMD hardware running VirtualBox. Windows Server has no native ARM64 build, so running this lab on an Apple Silicon MacBook meant none of the standard steps worked out of the box — from VM creation through to networking.

This project documents the full build **and** the architecture-specific problems that came up along the way: firmware mismatches between ARM64 and x86_64 virtual machines, OOBE setup failures under emulation, and a networking conflict between UTM's built-in DHCP and the domain controller's own DHCP service — none of which are covered in standard AD lab tutorials.



