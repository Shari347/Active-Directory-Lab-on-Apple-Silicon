# Active-Directory-Lab-on-Apple-Silicon
Deploying Windows Server 2019 via x86_64 Emulation (inspired by Josh Madakor's version with a twist)

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

A visual of the whole project and how each componenet is structured

<img width="1384" height="823" alt="image" src="https://github.com/user-attachments/assets/c1ec4788-67aa-4912-9286-eae89939baf6" />


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

**Networking (add two adapters by right-clicking one of the give adapters)**
- NIC1: Network Mode = **Shared Network** (internet-facing)
<img width="802" height="180" alt="image" src="https://github.com/user-attachments/assets/f33f08b0-bad9-48b0-ac16-4e3c65cb929a" />

- NIC2: Network Mode = **Host Only** (internal network)
<img width="802" height="226" alt="image" src="https://github.com/user-attachments/assets/c549ccfc-c4c2-4183-8cd1-dfd40997f46b" />

- Emulated Network Card: Intel Gigabit Ethernet (e1000) on both

## Step 2: Booting Up the VM

**Before booting, open VM Settings**
- Confirm under System: Architecture = **x86_64**, not ARM64

**🔧 Troubleshooting:** If you hit the firmware error *"QEMU error: combined size of system firmware exceeds 8388608 bytes"* →
Settings > QEMU tab > uncheck UEFI Boot, Save, reopen, check it again, Save.

**Boot the VM**

If it lands on `Shell>` (UEFI shell) instead of Windows Setup:
- Enter `fs0:` (try `fs1:` if that's empty)
- `ls` — confirm you see an **EFI** folder
- `cd EFI\BOOT` → Enter
- `BOOTX64.EFI` → Enter
- The instant "Press any key to boot from CD or DVD" appears, press a key immediately

**🔧 Troubleshooting:** If you hit the license terms error *"Windows cannot find the Microsoft Software License Terms"* → remove any second ISO (like utm-guest-tools) from a second CD/DVD drive, keep only `SERVER2019.iso` attached, restart, repeat the boot steps above if needed.

**Windows Server Setup**
- Language/keyboard → Next → **Install Now**
- **Windows Server 2019 Standard (Desktop Experience)**
- Accept license
- **Custom: Install Windows only (advanced)**
- Select unallocated disk → Next
- Wait through install (slow under emulation, reboots itself)
- Set local Administrator password when prompted

## Step 3: Configuring IP Addressing

There's one NIC dedicated to the internet and one for the internal network.

The external one doesn't need much attention — it'll automatically get an IP address from your home router. The internal one needs to be configured manually.

**1.Open Network Connections**
Right-click the Start button → Network Connections, or Control Panel > Network and Sharing Center > Change adapter settings.
<img width="858" height="452" alt="image" src="https://github.com/user-attachments/assets/f5f3a85d-f81b-4d21-b070-24b7e99335e7" />


**2.Identify the two NICs**
You'll see two adapters, likely named "Ethernet" and "Ethernet 2." To tell them apart, click each one → Details and check:
- The one with an **IPv4 Default Gateway** listed (e.g. `192.168.64.1`) = your **Internet-facing NIC** (Shared Network/NAT). Leave this one on DHCP, don't touch it.
<img width="2490" height="1752" alt="image" src="https://github.com/user-attachments/assets/e4908c2a-296d-4f34-832a-9740004b56b7" />

- The one with **no Default Gateway** listed = your **Internal NIC** (Host-Only). This is the one you configure.
<img width="2484" height="1758" alt="image" src="https://github.com/user-attachments/assets/1a92d04f-35bb-4262-9eb6-7431f3b777c2" />


Once you find which is which, rename the external one to **Internet** and the internal one to **Internal** (just names, to easily tell them apart later).

**3.Configure the Internal NIC**
1. Right-click the Internal NIC → **Properties**.
2. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
<img width="1730" height="1746" alt="image" src="https://github.com/user-attachments/assets/5e18fbb5-77f3-4a10-bb4c-d40a08437e38" />

3. Choose **Use the following IP address**:
   - IP address: `172.16.0.1`
   - Subnet mask: `255.255.255.0`
   - Default gateway: leave blank
4. Choose **Use the following DNS server address**:
   - Preferred DNS server: `127.0.0.1` (a loopback address referring to itself — the DC's own IP would also work here) as it whould look like whats below 
   <img width="2508" height="1774" alt="image" src="https://github.com/user-attachments/assets/3448e4f0-aeb8-4a8c-bf1c-86077288af8b" />

5. Click **OK**, then **Close**.

## Step 4: Install Active Directory Domain Services (AD DS) and Create a Domain

**Install the AD DS role**
1. Open **Server Manager** (should auto-launch, or Start menu).
2. Click **Manage** (top right) → **Add Roles and Features**.
3. Click **Next** through "Before You Begin."
4. Installation type: **Role-based or feature-based installation** → Next.
5. Server selection: leave the local server selected → Next.
<img width="2428" height="1706" alt="image" src="https://github.com/user-attachments/assets/42e43b74-1aa7-4269-aac8-764fb353a84c" />

6. Server roles: check **Active Directory Domain Services**.
<img width="790" height="576" alt="image" src="https://github.com/user-attachments/assets/3d429fa0-d8f3-426d-8d1c-8cf489c05657" />

7. A popup will ask to add required features (like RSAT tools) — click **Add Features** → Next.
8. Skip the Features page → Next.
9. Skip the AD DS info page → Next.
10. Confirm and click **Install**.
<img width="615" height="466" alt="image" src="https://github.com/user-attachments/assets/e3207351-93fb-4b3e-80cf-3984f724e0b3" />

11. Wait for it to finish (don't close the wizard, just let it run).

**Promote the server to a domain controller**
1. Once install finishes, click **"Promote this server to a domain controller"** (a link right in the results screen — or via the yellow flag notification icon at the top of Server Manager if you missed it).
<img width="297" height="245" alt="image" src="https://github.com/user-attachments/assets/dffb4d6e-1dbd-40d5-b55c-a416ffb3878b" />

2. Deployment Configuration: select **Add a new forest**.
3. Root domain name: type `mydomain.com` → Next.
<img width="580" height="430" alt="image" src="https://github.com/user-attachments/assets/220a61af-26f6-451b-a4c6-f47b6a2ba666" />

4. Domain Controller Options:
   - Forest/Domain functional level: leave default (usually fine as Windows Server 2016 or higher)
   - Ensure **DNS Server** is checked (it should be by default)
   - Set a **Directory Services Restore Mode (DSRM) password** — write this down somewhere safe(it can be the same password), it's separate from your admin password
   - Next
   <img width="580" height="429" alt="image" src="https://github.com/user-attachments/assets/6b7e5f50-b326-4556-9d34-4303e2c04355" />

5. DNS Options: you may see a warning about delegation — ignore it, click Next.
6. NetBIOS domain name: it'll auto-fill (likely `MYDOMAIN`) — leave as-is → Next.
7. Paths: leave default database/log/SYSVOL locations → Next.
8. Review Options: confirm everything looks right (domain name, DNS, etc.) → Next.
9. Prerequisites Check: it'll run a check — some warnings are normal/expected (like DNS delegation), as long as there's no red error blocking you, click **Install**.
10. The server will automatically **reboot** once promotion completes.

**After reboot**
- Log back in — you'll now log in as `MYDOMAIN\Administrator` instead of just `Administrator`, since this machine is now a domain controller for `mydomain.com`.
- Server Manager should show AD DS and DNS roles now active.
<img width="580" height="429" alt="image" src="https://github.com/user-attachments/assets/9458716b-4a6c-4f28-9e29-571d5c0737d9" />

## Step 5: Create a Domain Admin Account

**Create a dedicated Domain Admin account**
1. Open Server Manager → **Tools** → **Active Directory Users and Computers**.
2. Right-click your domain (`mydomain.com`) in the left pane → **New** → **Organizational Unit**.
<img width="502" height="402" alt="image" src="https://github.com/user-attachments/assets/00557e9e-7e5a-481b-bb2f-395b0364461a" />

3. Name it something like `Admins` → OK (leave "Protect from accidental deletion" checked).
<img width="335" height="293" alt="image" src="https://github.com/user-attachments/assets/4009c3d6-9e50-4457-bc49-a847aaf40c22" />

4. Right-click the new **Admins** OU → **New** → **User**.
<img width="335" height="293" alt="image" src="https://github.com/user-attachments/assets/287b8a52-ec6a-474c-9fa7-005e37ae9a1e" />

5. Fill in a first/last name and a logon name for yourself (e.g. `shari-admin`) → Next.
6. Set a password, uncheck "User must change password at next logon" if you don't want that hassle, check "Password never expires" for lab convenience → Next → Finish.
7. Right-click the new user you just created → **Properties**.
8. Go to the **Member Of** tab → **Add**.
9. Type `Domain Admins` → Check Names → OK → Apply → OK.
<img width="363" height="401" alt="image" src="https://github.com/user-attachments/assets/2279b4b3-e7c8-4c25-9f0d-eae5f4fd543b" />

10. Sign out of the built-in Administrator account.
11. At the login screen, choose **Other User**, log in as `MYDOMAIN\your-new-username` with the password you set.

## Step 6: Install and Set Up RAS/NAT

RAS/NAT is what actually connects your two isolated network segments together. Without it, Client1 would be stuck — it can talk to the DC (same internal network), but it has zero route to the actual internet.

**Install the Remote Access role (Routing)**
1. Open Server Manager → **Manage** → **Add Roles and Features**.
2. Next through "Before You Begin" → Role-based install → Next → local server selected → Next.
3. Check **Remote Access**.
<img width="601" height="429" alt="image" src="https://github.com/user-attachments/assets/dba681fd-5a19-491c-ad66-f6245f0fa528" />

4. Click **Add Features** if prompted → Next.
5. Skip Features page → Next.
6. Skip the Remote Access info page → Next.
7. On **Role Services**, check **Routing** (this auto-selects DirectAccess and VPN (RAS) too — leave those checked, you just need Routing but they come bundled).
<img width="603" height="432" alt="image" src="https://github.com/user-attachments/assets/3a955ed9-b626-44fe-be1c-ecfe979ff7bf" />

8. Click **Add Features** if prompted → Next.
9. Confirm → **Install**.
<img width="602" height="432" alt="image" src="https://github.com/user-attachments/assets/45890050-ee09-408b-9fa0-8358554d7e1f" />

10. Wait for it to finish → **Close**.

**Configure Routing and Remote Access**
1. In Server Manager, go to **Tools** → **Routing and Remote Access**.
2. In the left pane, right-click your server name (should show a red down-arrow icon, meaning not yet configured) → **Configure and Enable Routing and Remote Access**.
3. Wizard opens → Next.
<img width="2536" height="1768" alt="image" src="https://github.com/user-attachments/assets/7c15795c-505e-4c7d-aa14-f13bb366ab0e" />

4. Select **Network address translation (NAT)** → Next.
5. Choose the **public interface** — this is your Internet-facing NIC (the one with the gateway, on Shared Network/NAT) → make sure "Enable NAT on this interface" stays checked → Next.
   - If it doesn't pop up, close the Routing and Remote Access tab and reopen it — it should fix itself.
   <img width="2540" height="1778" alt="image" src="https://github.com/user-attachments/assets/78f859a9-5945-44a8-af3a-d7fd7dbf47bd" />

6. Finish.
7. It'll start the service — you should see the server icon change to a green arrow, meaning RAS/NAT is now running.

**Verify NAT config**
1. Still in Routing and Remote Access, expand your server → **IPv4** → **NAT**.
2. You should see two interfaces listed: your Internet-facing NIC marked as "Public interface connected to the Internet," and your Internal NIC marked as "Private interface connected to private network."

## Step 7: Setting Up a DHCP Server on Our Domain Controller

DHCP is what actually hands out IP addresses automatically, so you're not manually typing in an IP on every machine that joins the network.

**Install the DHCP Server role**
1. Server Manager → **Manage** → **Add Roles and Features**.
2. Next through "Before You Begin" → Role-based install → Next → local server selected → Next.
3. Check **DHCP Server**.
4. Click **Add Features** when prompted → Next.
5. Skip Features page → Next.
6. Skip the DHCP info page → Next.
7. Confirm → **Install**.
<img width="600" height="426" alt="image" src="https://github.com/user-attachments/assets/21b1e6a3-dad8-4402-84f9-13db70ccc0c9" />

8. Wait for it to finish.

**Complete DHCP post-install configuration**
1. Click the **yellow flag** notification icon.
2. Click **"Complete DHCP configuration"**.
3. Next on the intro screen.
4. Use current credentials (your Domain Admin account) → Commit.
5. Close.

**Create the DHCP scope**
1. Server Manager → **Tools** → **DHCP**.
2. Expand your server → **IPv4** → right-click **IPv4** → **New Scope**.
<img width="883" height="591" alt="image" src="https://github.com/user-attachments/assets/a01dd829-4928-4530-b2cb-bac2884d13e7" />

3. Scope Wizard → Next.
4. Name it something like `Internal Scope` → Next.
5. IP Address Range:
   - Start IP: `172.16.0.100`
   - End IP: `172.16.0.200`
   - Subnet mask: `255.255.255.0` → Next
   
<img width="669" height="573" alt="image" src="https://github.com/user-attachments/assets/c3da5b51-f11c-4ac1-8293-2a27715641c7" />

6. Add Exclusions: skip → Next.
7. Lease Duration (how long a client should keep that IP): leave default → Next.
8. Configure DHCP Options: **Yes, I want to configure these options now** → Next.
9. Router (Default Gateway): `172.16.0.1` → Add → Next.
10. Domain Name and DNS Servers:
    - Parent domain: `mydomain.com`
    - DNS server: `172.16.0.1` → Add → Next
<img width="637" height="569" alt="image" src="https://github.com/user-attachments/assets/c1f23e77-cac2-434e-b155-e96efcf9ef98" />

EXTRA INFO: When Client1 wants to reach something local — like the DC itself, for domain login or DNS — it doesn't need a gateway, it just sends directly on the `172.16.0.x` network. But when Client1 wants to reach something outside that network (like google.com), it has no idea how to get there. So it sends that traffic to whatever's configured as its default gateway, and lets that device figure out the rest.

11. WINS Servers: skip → Next.
12. Activate Scope: **Yes, I want to activate this scope now** → Next.
13. Finish.
14. Right-click the server (or the domain itself) in the DHCP console and click Authorize, then right-click it again and click **Refresh** so everything updates and the scope is visible.
<img width="2532" height="1758" alt="image" src="https://github.com/user-attachments/assets/6c109116-ec78-420c-b0b1-108b1e9dc0dd" />

## Step 8: Disabling IE Enhanced Security Configuration to Browse the Internet

1. Open **Server Manager**.
2. Click **Local Server** in the left sidebar.
3. Find **IE Enhanced Security Configuration** in the properties list on the right (it'll say "On").
<img width="1146" height="349" alt="image" src="https://github.com/user-attachments/assets/d8094b1a-07df-4046-87e9-e312b5ddcfa5" />

4. Click **On** next to it.
5. In the popup, set both **Administrators** and **Users** to **Off**.
<img width="316" height="344" alt="image" src="https://github.com/user-attachments/assets/49835fff-8d9d-4cb8-a5ca-65c1f1bd66b2" />

6. Click **OK**.
7. Close and reopen Internet Explorer (or restart it) for the change to take effect.

## Step 9: Bulk User Provisioning with PowerShell

Instead of manually creating a bunch of users, use a PowerShell script to add a whole batch of sample users to the server at once.

**Download the repo on the DC**
1. Open Internet Explorer on the DC (now that ESC is off) → go to `https://github.com/joshmadakor1/AD_PS`. i will also have the script in a folder with this repo
<img width="2994" height="1722" alt="image" src="https://github.com/user-attachments/assets/a8fddda3-2caa-44be-a160-42bb08724542" />

2. Click the green **Code** button → **Download ZIP**.
3. Save it — Desktop is easiest to find later.
4. Right-click the downloaded ZIP → **Extract All** → extract to Desktop.
<img width="855" height="453" alt="image" src="https://github.com/user-attachments/assets/8491c408-4b22-4ac1-9b98-c46a275c0a0a" />


**Add your name to the names list (optional)**
1. Open the extracted folder → open `names.txt`.
<img width="1439" height="1080" alt="image" src="https://github.com/user-attachments/assets/f354b60e-f23b-42d5-b01a-fe86b1c24637" />

2. Add your own name to the list if you want yourself included as one of the generated users → Save.
<img width="1443" height="1080" alt="image" src="https://github.com/user-attachments/assets/f6f186ee-462b-4596-9a1d-a15d33f91164" />


**Allow PowerShell scripts to run**
1. Click Start → type **PowerShell ISE** → right-click **Windows PowerShell ISE** → **Run as Administrator**.
<img width="1443" height="1079" alt="image" src="https://github.com/user-attachments/assets/f6490ea3-20dc-43db-b626-5f7e920ba6ed" />

2. In the console/command pane, type:
```powershell
   Set-ExecutionPolicy Unrestricted
```

3. Press Enter → type **A** (Yes to All) when it asks to confirm.
   - This temporarily allows unsigned scripts to run — normal for a lab, not something you'd do in production.

**Open and run the script**
1. In PowerShell ISE, **File > Open** → navigate to the extracted folder → open `1_CREATE_USERS.ps1`.
2. Check the top of the script — it should reference `names.txt` in the same folder, so as long as both files are together, it'll find it.
3. Click the green **Run** (play) button, or press **F5**.
4. Let it run — it'll loop through the names file and create a domain user for each one. With 1000+ names this can take a few minutes.

<img width="1286" height="296" alt="image" src="https://github.com/user-attachments/assets/8ea9ed13-3980-49da-8a91-66c27a527136" />
**🔧 Troubleshooting:** If you get an `ObjectNotFound` error for `names.txt`, run in PowerShell:

- cd C:\Users\YOURADMINNAMEDesktop\AD_PS-master (for exxample: cd C:\Users\s-hossain\Desktop\AD_PS-master)
- .\1_CREATE_USERS.ps1

This fixes a working-directory mismatch — the script should now find `names.txt` sitting in that folder.

**Verify**
1. Open **Active Directory Users and Computers**.
2. Check the OU the script creates users in (usually a new `_USERS` OU, or wherever the script's `-Path` parameter points).
3. You should see a long list of newly created user accounts.

<img width="1445" height="1009" alt="image" src="https://github.com/user-attachments/assets/1220445b-ba40-42b2-8c37-0c5bff1d44b8" />

---

### PowerShell Script Explanation

**Lines 1-3: Setup variables**
- `$PASSWORD_FOR_USERS = "Password1"` — sets one shared default password that every generated user account will get.
- `$USER_FIRST_LAST_LIST = Get-Content .\names.txt` — reads `names.txt` (must be in the same folder as the script) and loads every line into a list, one name per line.

**Line 6: Convert the password**
- `$password = ConvertTo-SecureString $PASSWORD_FOR_USERS -AsPlainText -Force` — Active Directory won't accept a plain text password directly, it needs it wrapped as a "secure string" object. This line converts the plain text password into the format AD requires.

**Line 7: Create the container**
- `New-ADOrganizationalUnit -Name _USERS -ProtectedFromAccidentalDeletion $false` — creates a new OU called `_USERS` where all these accounts will live. `ProtectedFromAccidentalDeletion $false` means it can be deleted later without extra confirmation steps.

**Line 9: The loop begins**
- `foreach ($n in $USER_FIRST_LAST_LIST) {` — starts a loop that runs once for every name in the list.

**Lines 10-12: Parse each name**
- `$first = $n.Split(" ")[0].ToLower()` —











































