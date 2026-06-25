# Windows Server Administration — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Someone who can use a Windows desktop but has never run a *server* — and wants to go all the way from "I just inserted the install media" to **operating Windows Server in production at a real company**: standing up an Active Directory domain, joining machines, locking it down with Group Policy, serving files and websites, virtualizing with Hyper-V, automating everything with PowerShell, monitoring it, backing it up, and hardening it against attackers. Every topic is explained in **prose first** — *what it is, why it exists, when and how you use it, the key options/parameters, the best practices, and the security implications* — and only then made concrete with **heavily-commented, runnable PowerShell** and, where the tool is GUI-driven, the exact click-path (Server Manager → Add Roles…). This is a *teaching* reference, not a terse cheat-sheet. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** Current as of **2026**. The reference release is **Windows Server 2025** (the newest Long-Term Servicing Channel / LTSC release, GA late 2024), with **Windows Server 2022** and **Windows Server 2019** still very common in production and fully supported. Key facts that shape everything below:
> - **Server Core vs Desktop Experience** are the two install options. *Server Core* has no Explorer shell, no MMC GUIs locally — you manage it remotely (Windows Admin Center, RSAT, PowerShell). It is the Microsoft-recommended default for infrastructure roles (smaller attack surface, fewer patches, less RAM). *Desktop Experience* gives you the familiar full GUI. **You cannot convert between them in 2025** — pick at install time.
> - **Windows Admin Center (WAC)** is Microsoft's modern browser-based management console (free, replaces much of the old MMC sprawl). It runs on a gateway and manages many servers, including Server Core, from one pane of glass.
> - **Two PowerShells coexist:** **Windows PowerShell 5.1** ships *in the box* (built on .NET Framework, `powershell.exe`) and runs the vast majority of server admin modules. **PowerShell 7.x** (built on .NET, cross-platform, `pwsh.exe`) is installed separately and is the modern language; some older RSAT/AD modules run under 5.1 via an implicit compatibility layer. Where it matters, this guide flags **⚡ PS 5.1 vs 7**.
> - **RSAT** (Remote Server Administration Tools) are the admin consoles (ADUC, DNS, DHCP, GPMC) you install on a Windows 11 management workstation to drive servers remotely.
> - **Azure Arc** extends Azure management (Policy, Update Manager, Defender) to on-prem Windows Servers — the modern "hybrid" story.
>
> Cross-references to sibling guides appear throughout: [PowerShell](POWERSHELL_GUIDE.md) (the language itself), [Windows CMD & Batch](WINDOWS_CMD_BATCH_GUIDE.md) (legacy shell), [Networking](NETWORKING_GUIDE.md) (IP/DNS/TLS fundamentals), [Nginx](NGINX_GUIDE.md) (reverse proxy in front of IIS), [Docker](DOCKER_GUIDE.md) (Windows containers), [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md) (SQL Server on Windows), and [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md) (the other half of most real fleets). Authoritative offline-checkable sources: `Get-Help`, `learn.microsoft.com` mirrors, and the on-box `*.adml`/baseline documentation from the Microsoft Security Compliance Toolkit.

---

## Table of Contents

1. [What Windows Server Is — Editions, Licensing, Server Core, and the Management Story](#1-what-windows-server-is--editions-licensing-server-core-and-the-management-story) **[B]**
2. [Installation & Initial Configuration](#2-installation--initial-configuration) **[B]**
3. [PowerShell for Administrators — Pipeline, Remoting, JEA, Running at Scale](#3-powershell-for-administrators--pipeline-remoting-jea-running-at-scale) **[B/I]**
4. [Users, Groups & Local Security — Accounts, NTFS vs Share, UAC, Least Privilege](#4-users-groups--local-security--accounts-ntfs-vs-share-uac-least-privilege) **[B/I]**
5. [Active Directory Domain Services — The Core](#5-active-directory-domain-services--the-core) **[I/A]**
6. [Managing AD — Objects at Scale, Delegation, and Group Policy in Depth](#6-managing-ad--objects-at-scale-delegation-and-group-policy-in-depth) **[I/A]**
7. [DNS & DHCP on Windows Server](#7-dns--dhcp-on-windows-server) **[I]**
8. [File Services — SMB, DFS, Quotas/FSRM, Storage Spaces, Dedup, VSS](#8-file-services--smb-dfs-quotasfsrm-storage-spaces-dedup-vss) **[I/A]**
9. [Web & App Hosting — IIS in Depth](#9-web--app-hosting--iis-in-depth) **[I/A]**
10. [Hyper-V Virtualization](#10-hyper-v-virtualization) **[I/A]**
11. [Storage & Disks — Partitioning, Storage Spaces, iSCSI, ReFS vs NTFS](#11-storage--disks--partitioning-storage-spaces-iscsi-refs-vs-ntfs) **[I/A]**
12. [Networking & Firewall on Windows Server](#12-networking--firewall-on-windows-server) **[I]**
13. [Security & Hardening](#13-security--hardening) **[A]**
14. [Monitoring & Logging](#14-monitoring--logging) **[I/A]**
15. [Backup & Disaster Recovery](#15-backup--disaster-recovery) **[I/A]**
16. [Automation & At-Scale Management — Scheduled Tasks, DSC, WAC, Azure Arc](#16-automation--at-scale-management--scheduled-tasks-dsc-wac-azure-arc) **[A]**
17. [Production Operations & Company-Level Practices](#17-production-operations--company-level-practices) **[A]**
18. [Gotchas & Best Practices](#18-gotchas--best-practices) **[I/A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What Windows Server Is — Editions, Licensing, Server Core, and the Management Story

### 1.1 What "Windows Server" actually is and why it exists **[B]**

**Windows Server is a separate operating system from Windows 11**, built on the same kernel but tuned, licensed, and supported for running *services* that many other computers depend on — not for one person browsing the web. The difference is not cosmetic. A desktop OS optimizes for foreground responsiveness, consumer features (Store, gaming, Cortana-era cruft), and a single interactive user. A server OS optimizes for **uptime, background services, many simultaneous remote users, and roles** — large, first-party capabilities like *Active Directory*, *DNS*, *DHCP*, *file sharing*, *IIS web hosting*, and *Hyper-V virtualization* that you "add" to the box.

The reason a business runs Windows Server rather than a pile of desktops is **centralization**. Instead of 500 laptops each with their own local accounts and settings, you run a few servers that hold *one* directory of users and groups (Active Directory), push *one* set of policies to every machine (Group Policy), store files in *one* governed place (file services), and authenticate everyone against *one* trusted authority. That central control — identity, policy, storage, and services — is the entire point.

A **role** is a major function the server provides to *other* computers (AD DS, DNS, DHCP, File and Storage Services, Web Server/IIS, Hyper-V). A **feature** is a supporting capability for the server *itself* (e.g., .NET Framework, Failover Clustering, Windows Server Backup, the BitLocker feature). You install roles and features; everything in this guide is, ultimately, the careful operation of a handful of roles.

### 1.2 Editions: Standard vs Datacenter (and the small-business past) **[B]**

For the mainstream LTSC product there are two editions, differing mainly by **virtualization rights** and a few advanced storage features — *not* by which roles you can install (both can be a domain controller, file server, web server, etc.):

| Edition | Who it's for | Virtualization rights | Notable extras |
|---|---|---|---|
| **Standard** | Most physical servers, light virtualization | License covers the host + **2** Windows Server VMs (OSEs) | All core roles |
| **Datacenter** | Highly virtualized hosts, private cloud | **Unlimited** Windows Server VMs on the licensed host | **Storage Spaces Direct (S2D)**, Software-Defined Networking, Shielded VMs, Storage Replica (full) |

> **⚡ Version note:** The old **Essentials** edition (for very small businesses, ≤25 users, with a simplified dashboard) was effectively retired after Server 2019/2022; in the 2025 era small shops are pushed toward **Microsoft 365 / Azure AD (Entra ID)** or a plain Standard box. There is also a **Windows Server (Annual Channel for containers)** for container hosts, distinct from the LTSC desktop/server you'll mostly run.

### 1.3 Licensing in one honest paragraph **[B]**

Windows Server is licensed by **physical cores** (core-based licensing): you must license **all** physical cores in the server, with a **minimum of 16 cores per server (and 8 per processor)** — sold in 2-core packs. On top of the server license, **every user or device that *accesses* the server needs a Client Access License (CAL)** — either a User CAL (per person, any device) or Device CAL (per device, any person). Specific roles (Remote Desktop Services, AD Rights Management) need *additional* RDS/RMS CALs. CALs are a paperwork/compliance matter, not technically enforced for most roles, but they are real and audited. **Takeaway:** budget for (cores × edition) **plus** CALs; in a virtualized environment, Datacenter's unlimited-VM rights usually win past ~10–14 VMs per host.

### 1.4 Server Core vs Desktop Experience — the most important early choice **[B]**

When you install Windows Server 2025 you choose one of two installation types, and **you cannot switch later** (this changed years ago — gone is the old "remove the GUI" trick):

- **Desktop Experience** — the full GUI you know: Explorer, the Start menu, MMC consoles, browsers, the works. Easiest to learn on; biggest attack surface; most patches (every GUI/Edge/font CVE applies); most RAM and disk. Good for app servers whose vendor only supports a GUI, RDS session hosts, and learning.
- **Server Core** — **no desktop shell at all.** When you log on locally you get a single command prompt (and `sconfig`). There is no Explorer, no `mmc.exe`, no local Server Manager GUI. You administer it **remotely** with Windows Admin Center, RSAT consoles, and PowerShell remoting. In return you get a **~40% smaller footprint, far fewer reboots/patches, and a dramatically smaller attack surface.** Microsoft recommends Core as the default for infrastructure roles (DC, DNS, DHCP, file, Hyper-V).

| | Server Core | Desktop Experience |
|---|---|---|
| Local GUI | None (CLI + `sconfig`) | Full Explorer/MMC |
| Footprint / RAM | Small | Large |
| Patch surface / reboots | Minimal | Maximal |
| Management | Remote (WAC/RSAT/PS) | Local or remote |
| Best for | DC, DNS, DHCP, file, Hyper-V, production infra | App servers needing a GUI, RDS, labs/learning |

**Practical advice for a beginner:** *learn* on Desktop Experience (so you can see the consoles), but understand that real production infrastructure increasingly runs on Core managed from a Windows 11 admin workstation. Everything in this guide is given as PowerShell **first** precisely so it works identically on Core.

### 1.5 The management story — how you actually administer servers **[B]**

You rarely sit at a server's physical console. The four pillars:

1. **Server Manager** (Desktop Experience only) — the dashboard that opens on logon. From here you *Add Roles and Features*, monitor role health, and (crucially) **manage *other* servers remotely** by adding them to a server pool. One Server Manager can drive a whole farm.
2. **Windows Admin Center (WAC)** — a modern, browser-based, extensible console you install on a **gateway** machine. Point a browser at it, and you get tools for almost every role (files, certificates, events, registry, PowerShell, updates, VMs) across all your servers — *including Server Core*. This is the strategic direction and the friendliest way to manage Core.
3. **RSAT (Remote Server Administration Tools)** — the classic MMC snap-ins (Active Directory Users and Computers "ADUC", DNS Manager, DHCP, Group Policy Management Console "GPMC", DFS Management). You install these as **optional features on Windows 11** and point them at your servers. This is where deep AD/GPO work still happens.
4. **PowerShell remoting** — `Enter-PSSession` / `Invoke-Command` over **WinRM**, the engine that lets you run cmdlets on any server (or hundreds at once) from your desk. Covered in depth in §3.

```powershell
# See which roles/features are installed on THIS server (works on Core and Desktop).
Get-WindowsFeature | Where-Object Installed | Select-Object Name, DisplayName

# Add the Server Manager remote-management pool view from your admin box?
# (You add servers in the GUI: Server Manager -> Manage -> Add Servers -> by AD/DNS/Import.)

# Install Windows Admin Center on a gateway server (download the MSI offline first):
#   msiexec /i WindowsAdminCenter.msi /qn SME_PORT=443 SSL_CERTIFICATE_OPTION=generate
# Then browse to https://<gateway>:443 and add target servers.
```

> **Best practice / security:** Manage servers from a hardened **Privileged Access Workstation (PAW)** — a locked-down admin machine — never by RDP-ing in with a Domain Admin account and browsing the web. The tiered-admin model in §17 explains why this matters enormously.

---

## 2. Installation & Initial Configuration

### 2.1 The install itself **[B]**

Installation is wizard-driven and short. Boot from the ISO/USB, choose language, click **Install now**, pick the **edition + install type** (Standard/Datacenter × Core/Desktop — §1.4, remember you can't change install type later), accept the license, choose **Custom (clean install)**, select/partition the target disk, and wait through the reboots. On first logon you set the built-in **Administrator** password. That's it — but a freshly installed server is *unconfigured and not production-ready*; the value is in what you do next.

A clean server has: a random computer name, a DHCP address, default time zone, no roles, Windows Update on default settings, and (Desktop edition) an aggressively locked-down IE/Edge "Enhanced Security Configuration." Your **initial configuration checklist** is: rename → static IP → time zone → updates → activate → (optionally) join domain → install roles.

### 2.2 `sconfig` — the text menu that runs the show on Core **[B]**

On **Server Core**, your first-stop tool is **`sconfig`** (Server Configuration), a numbered text menu that wraps the most common setup tasks: rename the computer, join a domain/workgroup, configure network (static IP/DNS), set the date/time zone, enable Remote Desktop, configure Windows Update, and reboot/shutdown. Just run `sconfig` at the prompt and pick numbers. It exists so you can do day-zero setup on a GUI-less box without memorizing cmdlets. (It's also available on Desktop Experience.)

```text
# Typical sconfig menu (run: sconfig)
 1) Domain/Workgroup          7) Remote Desktop
 2) Computer Name             8) Network Settings
 3) Add Local Administrator   9) Date and Time
 4) Configure Remote Mgmt    10) Telemetry settings
 5) Windows Update Settings  11) Windows Activation
 6) Download/Install Updates 12) Log Off User ...
                             13) Restart  14) Shut Down  15) Exit to CLI
```

Everything `sconfig` does is also a cmdlet — and for repeatability you should script it. Here is the **whole initial config as PowerShell**, which is what you'd actually do in production (and the only option in fully automated builds):

```powershell
# --- 1. Rename the computer (requires a reboot) ---
Rename-Computer -NewName "DC01" -Restart    # -Restart reboots immediately

# --- 2. Set a STATIC IP (servers must NOT use DHCP; clients rely on a fixed address) ---
# Find your adapter's name/index first:
Get-NetAdapter | Format-Table Name, InterfaceIndex, Status, LinkSpeed

# Remove any existing DHCP address, then assign static IPv4:
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 10.0.0.10 -PrefixLength 24 -DefaultGateway 10.0.0.1

# DNS: a domain controller points at ITSELF (or 127.0.0.1); members point at the DCs.
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.10,10.0.0.11

# --- 3. Time zone + time sync (AD breaks if clocks drift > 5 minutes — Kerberos!) ---
Set-TimeZone -Name "GMT Standard Time"       # list options: Get-TimeZone -ListAvailable
w32tm /resync                                # force a time sync now

# --- 4. Windows Update (check + install; see §13 for WSUS at scale) ---
# Built-in module on 2025; otherwise use the PSWindowsUpdate module offline.
# Install-Module PSWindowsUpdate ; Get-WindowsUpdate -Install -AcceptAll -AutoReboot

# --- 5. Activate (with a product key or via KMS/ADBA in an enterprise) ---
slmgr.vbs /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX   # install key
slmgr.vbs /ato                                  # activate online
slmgr.vbs /dlv                                  # show licensing detail
```

> **⚡ Version note:** `New-NetIPAddress`, `Set-DnsClientServerAddress`, and `Set-TimeZone` are modern cmdlets present since Server 2012/2016 and current in 2025. The older `netsh interface ip set address` still works but is legacy — prefer the cmdlets.

### 2.3 Why a server needs a static IP, correct DNS, and correct time **[B]**

Three settings cause more first-week pain than anything else:

- **Static IP.** Other machines find this server by address; if DHCP moves it, DNS records and client configs break. *Always static for servers* (or a DHCP **reservation** so it's effectively fixed).
- **DNS pointing at your domain controllers, never at a public resolver.** This is the #1 cause of "can't join the domain." A domain member must resolve `_ldap._tcp.<domain>` SRV records, which only your **internal AD-integrated DNS** serves. Pointing a member at `8.8.8.8` will make domain logon and Group Policy silently fail.
- **Accurate time.** Active Directory's Kerberos authentication rejects tickets when the clock skew exceeds **5 minutes** by default. The PDC-Emulator FSMO role (§5) is the domain's time root; everything else syncs down from it. A drifting clock = mysterious "access denied" everywhere.

### 2.4 Enable remote management early **[B]**

So you can stop using the physical console:

```powershell
# Enable PowerShell remoting (WinRM listener, firewall rules). Usually on by default
# on Server, but be explicit. Run in an elevated session.
Enable-PSRemoting -Force

# Enable Remote Desktop (for occasional GUI/RDP access)
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Confirm WinRM is listening
Test-WSMan -ComputerName localhost
```

---

## 3. PowerShell for Administrators — Pipeline, Remoting, JEA, Running at Scale

> Deep language coverage lives in the [PowerShell](POWERSHELL_GUIDE.md) guide; this section is the *server-administration* slice — the parts you cannot run a Windows fleet without. If you only learn one tool in this whole guide, learn this one. The [Windows CMD & Batch](WINDOWS_CMD_BATCH_GUIDE.md) guide covers the legacy shell you'll still occasionally meet.

### 3.1 Why PowerShell, and the object pipeline **[B]**

PowerShell is *the* administration language for Windows. Its defining idea: **the pipeline carries .NET objects, not text.** In bash, `ls | grep ...` passes strings and you parse columns by hand. In PowerShell, `Get-Process | Where-Object CPU -gt 100` passes *process objects* with typed properties (`.CPU`, `.Id`, `.Name`), so you filter, sort, group, and format by **property name** with zero parsing. This is why server automation in PowerShell is so much more robust than batch scripts: you never scrape text.

```powershell
# Verb-Noun is the universal cmdlet shape. Discover anything:
Get-Command -Noun Service           # all *-Service cmdlets
Get-Help Get-Service -Examples      # worked examples, offline
Get-Service | Get-Member            # the REAL way to learn an object's properties/methods

# The pipeline in action — find big, running services and stop one safely:
Get-Service |
  Where-Object { $_.Status -eq 'Running' -and $_.StartType -eq 'Automatic' } |
  Sort-Object DisplayName |
  Select-Object Name, DisplayName, Status

Restart-Service -Name 'Spooler' -WhatIf   # -WhatIf shows what WOULD happen, changes nothing
```

> **Best practice:** `-WhatIf` and `-Confirm` are your seatbelts. Run a destructive command with `-WhatIf` first **every time** until you trust it. Many cmdlets support them automatically.

### 3.2 PowerShell 5.1 vs 7.x for admins **[B/I]**

| | Windows PowerShell **5.1** | PowerShell **7.x** |
|---|---|---|
| Ships in-box? | Yes (`powershell.exe`) | No — install separately (`pwsh.exe`) |
| Runtime | .NET Framework | .NET (cross-platform) |
| Module coverage | Broadest for legacy server modules | Modern; some old modules via WinPSCompatSession |
| Use it for | Anything that only ships a 5.1 module | New scripts, cross-platform, parallel `ForEach-Object` |

> **⚡ PS 5.1 vs 7:** Most RSAT/AD/DNS/DHCP modules are *built for 5.1*. PowerShell 7 transparently loads many of them through an **implicit compatibility session** (it spins up a hidden 5.1 process). It usually "just works," but if a cmdlet behaves oddly in 7, retry it in `powershell.exe`. New automation should target 7 where possible (notably `ForEach-Object -Parallel`).

### 3.3 Remoting — WinRM, `Invoke-Command`, and `Enter-PSSession` **[B/I]**

**Remoting is how you run commands on other servers from your desk.** It rides **WinRM** (Windows Remote Management, the WS-Man protocol) over **TCP 5985 (HTTP)** / **5986 (HTTPS)**. Two everyday shapes:

```powershell
# Interactive: drop INTO a remote server's shell, run a few things, exit.
Enter-PSSession -ComputerName DC01 -Credential (Get-Credential)
#  ... you are now "on" DC01 ...
Exit-PSSession

# Fan-out: run ONE command on MANY servers at once and get objects back, labelled by source.
Invoke-Command -ComputerName SRV01,SRV02,SRV03 -ScriptBlock {
    Get-Service Spooler | Select-Object Status, StartType
}
# Each result carries a PSComputerName property so you know which box it came from.

# Reusable, efficient sessions (avoid re-authenticating for every call):
$s = New-PSSession -ComputerName SRV01,SRV02
Invoke-Command -Session $s -ScriptBlock { (Get-CimInstance Win32_OperatingSystem).LastBootUpTime }
Remove-PSSession $s
```

In a **domain**, remoting authenticates with Kerberos automatically (no password prompt for your own credentials). Across a **workgroup/untrusted boundary**, you must add the target to `TrustedHosts` and supply credentials, ideally over **HTTPS (5986)** with a real certificate so traffic is encrypted and the server is authenticated.

```powershell
# Workgroup/untrusted: trust a specific host (do NOT use "*").
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 'SRV05.contoso.local' -Concatenate
```

> **Security:** WinRM over HTTP (5985) is fine *inside* a domain (the payload is still encrypted by the negotiated auth), but for cross-trust or DMZ scenarios use **5986/HTTPS**. Lock down *who* can remote in with the `Remote Management Users` group, and prefer **JEA** (next) over handing out full admin.

### 3.4 `CIM`/`WMI` — querying the machine's instrumentation **[I]**

WMI/CIM is Windows's vast inventory database — hardware, OS, disks, network, installed software. Use the **CIM cmdlets** (`Get-CimInstance`), not the deprecated `Get-WmiObject`. CIM uses WinRM and returns clean objects.

```powershell
Get-CimInstance Win32_OperatingSystem  | Select-Object Caption, Version, LastBootUpTime
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" |
  Select-Object DeviceID,
    @{n='FreeGB';e={[math]::Round($_.FreeSpace/1GB,1)}},
    @{n='SizeGB';e={[math]::Round($_.Size/1GB,1)}}
Get-CimInstance Win32_ComputerSystem   | Select-Object Manufacturer, Model, TotalPhysicalMemory
```

### 3.5 Just Enough Administration (JEA) — least-privilege remoting **[I/A]**

**JEA solves a dangerous problem:** you want a helpdesk tech to be able to *restart a print spooler* on a server, but you do **not** want to make them a local admin. JEA lets you publish a **constrained PowerShell endpoint** where a non-admin connects and can run **only an allow-listed set of cmdlets/parameters**, executed under a temporary, privileged **virtual account** — with full transcription/logging. It is role-based, least-privilege remote administration, and it's a cornerstone of a hardened environment.

Two files define it: a **Role Capability** file (`.psrc` — *what* commands are allowed) and a **Session Configuration** file (`.pssc` — *who* maps to which role, plus run-as and logging settings).

```powershell
# 1. Create a module to hold the role capability, then a .psrc listing allowed commands.
New-Item -Path "C:\Program Files\WindowsPowerShell\Modules\HelpdeskJEA\RoleCapabilities" `
  -ItemType Directory -Force
New-PSRoleCapabilityFile `
  -Path "C:\Program Files\WindowsPowerShell\Modules\HelpdeskJEA\RoleCapabilities\Helpdesk.psrc" `
  -VisibleCmdlets 'Restart-Service', @{ Name='Restart-Service'; Parameters=@{ Name='Name'; ValidateSet='Spooler' } }, `
                  'Get-Service', 'Get-Process'

# 2. Create the session config: map a security group to that role, run as a virtual account, log everything.
New-PSSessionConfigurationFile -Path "C:\JEA\Helpdesk.pssc" `
  -SessionType RestrictedRemoteServer `        # locks the session to only what we allow
  -RunAsVirtualAccount `                        # commands run privileged, the USER stays non-admin
  -TranscriptDirectory "C:\JEA\Transcripts" `   # every session is recorded to disk
  -RoleDefinitions @{ 'CONTOSO\Helpdesk' = @{ RoleCapabilities = 'Helpdesk' } }

# 3. Register the endpoint (this restarts WinRM).
Register-PSSessionConfiguration -Name 'Helpdesk' -Path "C:\JEA\Helpdesk.pssc" -Force

# A helpdesk user then connects to the CONSTRAINED endpoint and can do nothing but the allowed commands:
#   Enter-PSSession -ComputerName SRV01 -ConfigurationName Helpdesk -Credential (Get-Credential)
```

> **Why this matters at company scale:** JEA lets you delegate *narrow* operational tasks to junior staff and applications without ever granting standing admin rights — directly supporting the least-privilege and tiered-admin models in §13 and §17.

### 3.6 Running tasks at scale — patterns **[I]**

```powershell
# Define the fleet once (or pull it from AD — see §6).
$servers = (Get-ADComputer -Filter 'OperatingSystem -like "*Server*"').Name

# Sequential fan-out with error handling per host:
Invoke-Command -ComputerName $servers -ErrorAction SilentlyContinue -ScriptBlock {
    [PSCustomObject]@{
        Uptime = (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
        Pending = Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'
    }
} | Sort-Object PSComputerName | Format-Table PSComputerName, Uptime, Pending

# PowerShell 7 local parallelism (e.g., pinging/collecting from many hosts):
$servers | ForEach-Object -Parallel { Test-Connection $_ -Count 1 -Quiet } -ThrottleLimit 20
```

> **Best practice:** Build a **report, not a guess.** Emit `[PSCustomObject]` rows from your fan-out scripts, then `Sort`, `Where`, `Export-Csv`. Treat your fleet like a queryable database.

---

## 4. Users, Groups & Local Security — Accounts, NTFS vs Share, UAC, Least Privilege

### 4.1 Local accounts vs domain accounts **[B]**

Every Windows machine has a **local account database (SAM)** with a built-in **Administrator** and built-in groups. A standalone server uses these. The moment you **join a domain** (§5), the far more powerful **domain accounts** appear — central identities usable on every domain machine. Best practice in a domain: **almost never use local accounts** for people; use domain accounts and reserve the local Administrator strictly for break-glass recovery (with its password randomized per-machine by **LAPS**, §13).

```powershell
# Local account management (standalone server or local break-glass account):
Get-LocalUser
New-LocalUser -Name "svc_backup" -Description "Backup service account" -NoPassword:$false `
  -Password (Read-Host -AsSecureString "Password")
Add-LocalGroupMember -Group "Administrators" -Member "svc_backup"
Get-LocalGroupMember -Group "Administrators"
```

### 4.2 Built-in groups you must know **[B]**

| Group | What membership grants |
|---|---|
| **Administrators** | Full control of the machine. Keep tiny. |
| **Users** | Standard, safe day-to-day rights. |
| **Remote Desktop Users** | Allowed to RDP in (without being admin). |
| **Remote Management Users** | Allowed to use WinRM/remoting. |
| **Backup Operators** | Can back up/restore *bypassing* file ACLs (powerful — treat as admin-adjacent). |
| **Domain Admins** (domain) | Full control of the whole **domain**. The crown jewels. |
| **Enterprise Admins** (forest) | Full control of the **entire forest**. Use almost never. |

> **Security:** **Domain Admins / Enterprise Admins are the keys to the kingdom.** Membership should be a handful of named, dedicated admin accounts (never your daily-driver account), protected by the tiered model and the **Protected Users** group (§13). An attacker who lands one Domain Admin token can own everything.

### 4.3 NTFS permissions vs Share permissions — the eternal confusion **[B/I]**

When you share a folder over the network, **two independent permission layers** apply, and the user's effective access is the **most restrictive of the two**:

- **Share permissions** govern access *over the network* to the share as a whole. They're coarse (Read / Change / Full Control) and don't apply when sitting at the console.
- **NTFS permissions (ACLs)** govern access to *files and folders themselves*, both locally and over the network, with fine granularity (Read, Write, Modify, Full Control, and special permissions), and they **inherit** down the tree.

The modern best practice: **set Share permissions to `Authenticated Users = Full Control`** (i.e., wide open at the share layer) **and do all real access control with NTFS permissions.** This avoids the trap of having to reconcile two permission models. NTFS is where you grant `Sales = Modify` on the `Sales` folder.

```powershell
# Create a share where NTFS does the real work:
New-Item -Path "D:\Shares\Sales" -ItemType Directory -Force
New-SmbShare -Name "Sales" -Path "D:\Shares\Sales" -FullAccess "Authenticated Users"

# Now set NTFS (the real gatekeeper). icacls is the workhorse:
icacls "D:\Shares\Sales" /grant "CONTOSO\Sales-RW:(OI)(CI)M"  # Modify, inherited to files+subfolders
icacls "D:\Shares\Sales" /grant "CONTOSO\Sales-RO:(OI)(CI)RX" # Read+Execute
icacls "D:\Shares\Sales" /inheritance:r                       # remove inherited perms when locking down
icacls "D:\Shares\Sales"                                      # AUDIT current ACL
```

> **Gotcha:** `(OI)` = object inherit (files), `(CI)` = container inherit (subfolders). Forget them and your permission applies only to the top folder. **Effective access = min(Share, NTFS)** — if a user "can't write" despite Modify on NTFS, check the Share layer (or vice-versa).

### 4.4 UAC — why even admins run de-elevated **[B]**

**User Account Control** means that even a member of Administrators runs with a *standard* token by default; privileged actions trigger an elevation prompt ("Run as administrator"), splitting your session into a filtered token and a full token. This limits the blast radius of malware running in your context. **Do not disable UAC** to "make things easier" — instead, open an elevated PowerShell/console deliberately when you need it. In scripts, detect elevation:

```powershell
# Am I elevated? (true/false)
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
 ).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

### 4.5 The principle of least privilege (PoLP) **[B/I]**

The single most important security idea in administration: **every account, service, and person should have the minimum rights needed and no more.** Concretely on Windows Server:

- People get **two** accounts: a normal user account for email/web, and a **separate admin account** used *only* for admin tasks, never for browsing.
- **Service accounts** run services — use **group Managed Service Accounts (gMSA)** so the password is auto-rotated by AD and never known to a human (§5/§13).
- Grant via **groups**, not individuals — "add to the `Sales-RW` group," never "ACL this one person." Memberships are auditable and revocable.
- Delegate narrow tasks via **JEA** (§3.5) and AD **delegation** (§6) instead of group membership where possible.

---

## 5. Active Directory Domain Services — The Core

> Active Directory is the heart of Windows Server in a company. It's also the topic beginners find most abstract, so this section builds the mental model carefully before any cmdlets. It ties directly to the [Networking](NETWORKING_GUIDE.md) guide (DNS, Kerberos, LDAP all ride the network you learned there).

### 5.1 What a directory service *is*, and why AD exists **[I]**

**Active Directory Domain Services (AD DS) is a centralized, replicated database of "who and what is in your organization" — users, computers, groups, printers — plus the authentication and policy machinery built on top of it.** Before AD, every server had its own user list; logging into 30 servers meant 30 separate accounts. AD replaces that with **one identity per person**, trusted everywhere in the organization. When you log into *any* domain-joined PC with your domain account, that PC asks a **domain controller** to verify you, and the DC issues you cryptographic tickets (Kerberos) you then present to file servers, web apps, and databases — single sign-on across the whole company.

Under the hood AD is an **LDAP** directory (the query protocol), uses **Kerberos** for authentication, and depends utterly on **DNS** for clients to *find* domain controllers. Those three protocols (LDAP, Kerberos, DNS) are the legs AD stands on.

### 5.2 The hierarchy — domain, tree, forest, OU **[I]**

These four words trip up every beginner. Precisely:

| Term | What it is | Analogy |
|---|---|---|
| **Domain** | The core unit: one replicated database of users/computers/groups sharing a name like `contoso.com`, one authentication boundary. | One company building |
| **Tree** | One or more domains sharing a *contiguous* DNS namespace (`contoso.com`, `eu.contoso.com`). | A street of buildings sharing a name |
| **Forest** | The **top-level security boundary** — one or more trees, sharing a schema and global catalog, with automatic trust between all domains. | The whole campus |
| **Organizational Unit (OU)** | A **container *inside* a domain** for organizing objects and **targeting Group Policy / delegating admin**. *Not* a security boundary. | Rooms/departments inside a building |

**Two ideas beginners must internalize:**
1. **The forest, not the domain, is the real security boundary.** If you need true isolation between two business units (e.g., for compliance), separate *forests* — not just separate OUs or domains.
2. **OUs are for administration, groups are for permissions.** You put the `Sales` users' *accounts* in a `Sales` **OU** (to apply policy and delegate), but you grant file access via a `Sales` **security group**. Don't confuse the two — a very common beginner error.

Most companies today run a **single domain, single forest** with a well-designed OU structure. The "multiple domains" era is largely over; simplicity wins.

### 5.3 Domain controllers and the role of DNS **[I]**

A **domain controller (DC)** is a server running the AD DS role that holds a **writable copy** of the domain database and answers authentication/LDAP requests. You always run **at least two DCs** per domain for redundancy — if your only DC dies, *no one can log in.* DCs **multi-master replicate**: a change on one DC propagates to all others (most changes within minutes via change notification within a site).

**DNS is not optional — it is how clients find DCs.** When a PC boots, it queries DNS for **SRV records** like `_ldap._tcp.dc._msdcs.contoso.com` to locate a domain controller. This is why AD DNS zones are almost always **AD-integrated** (stored in AD itself, replicated with it) and why a domain member's DNS must point at the DCs, never at a public resolver (§2.3). Misconfigured DNS is the cause of the overwhelming majority of "AD is broken" tickets.

### 5.4 Promoting the first domain controller **[I/A]**

"Promoting" a server means installing AD DS and turning it into a DC, creating a new forest in the process. The click-path is *Server Manager → Add Roles → Active Directory Domain Services → (flag) → Promote this server to a domain controller*. The scripted, repeatable way:

```powershell
# --- Step 1: Install the AD DS role and its management tools ---
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# --- Step 2: Promote to the FIRST DC of a BRAND-NEW forest ---
# This also installs DNS, sets the functional levels, and prompts for the DSRM password.
Install-ADDSForest `
  -DomainName "contoso.local" `                 # internal name; see naming note below
  -DomainNetbiosName "CONTOSO" `                 # the short pre-Windows-2000 name
  -ForestMode "WinThreshold" `                   # forest functional level (2016 schema; see note)
  -DomainMode "WinThreshold" `
  -InstallDns `                                  # stand up AD-integrated DNS
  -DatabasePath "C:\Windows\NTDS" `
  -SysvolPath   "C:\Windows\SYSVOL" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
  -Force
# The server reboots. After reboot it IS a domain controller for contoso.local.
```

> **⚡ Version note — functional levels & domain names:** Windows Server 2025 introduced a **new "Windows Server 2025" domain/forest functional level** (the first new level in years), unlocking features like the 32-bit-page-aligned **AD database** improvements and Kerberos enhancements. To raise to it, **all** DCs must run 2025. The `-ForestMode/-DomainMode` value `"WinThreshold"` = the 2016 level (a safe, widely compatible baseline); use the 2025 level only once every DC is upgraded. **Naming:** do **not** name your AD domain the same as your public website (`contoso.com`). Use a routable subdomain you own (`ad.contoso.com`) or a dedicated internal label — `.local` is legacy and discouraged for new builds because it complicates public certs; the modern recommendation is a delegated subdomain of a domain you actually own.

```powershell
# Add a SECOND DC to an EXISTING domain (redundancy — always do this):
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSDomainController `
  -DomainName "contoso.local" `
  -InstallDns `
  -Credential (Get-Credential "CONTOSO\Administrator") `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") -Force
```

### 5.5 Joining machines to the domain **[I]**

Once a domain exists, you join servers and workstations to it so they trust the central directory:

```powershell
# Join this computer to the domain and reboot. (Set DNS to a DC FIRST — §2.2!)
Add-Computer -DomainName "contoso.local" -Credential (Get-Credential "CONTOSO\joinacct") -Restart

# Verify the trust/secure channel afterwards:
Test-ComputerSecureChannel -Verbose
```

GUI path: *Settings → System → About → Domain or workgroup → Change → Domain*. After joining, you can log in with `CONTOSO\username` (or `username@contoso.local`) on that machine, and it appears under *Computers* in ADUC.

### 5.6 FSMO roles — the five single-master jobs **[I/A]**

AD is *multi*-master for most operations, but **five operations are too sensitive to allow conflicting simultaneous edits**, so exactly one DC holds each **FSMO (Flexible Single Master Operations)** role at a time. You must know these for troubleshooting and for the *transfer/seize* procedures during DC retirement or failure:

| FSMO role | Scope | What it does | If it's down… |
|---|---|---|---|
| **Schema Master** | Forest | Owns changes to the AD **schema** (object/attribute definitions). | Can't extend schema (rare). |
| **Domain Naming Master** | Forest | Owns adding/removing **domains** in the forest. | Can't add/remove domains (rare). |
| **PDC Emulator** | Domain | **Most important day-to-day:** authoritative **time source**, password-change priority, account-lockout processing, GPO editing target. | Time drift, login/lockout problems — fix fast. |
| **RID Master** | Domain | Hands out pools of **RIDs** (the unique part of every SID) to DCs. | Eventually can't create new objects. |
| **Infrastructure Master** | Domain | Tracks cross-domain object references. | Stale cross-domain group names. |

```powershell
# See who holds the FSMO roles (memorize this command):
Get-ADDomain  | Select-Object PDCEmulator, RIDMaster, InfrastructureMaster
Get-ADForest  | Select-Object SchemaMaster, DomainNamingMaster
# Or all five at once:
netdom query fsmo

# Gracefully TRANSFER a role to another DC (planned — e.g., retiring a DC):
Move-ADDirectoryServerOperationMasterRole -Identity "DC02" -OperationMasterRole PDCEmulator

# SEIZE a role (ONLY if the holder is permanently dead and will never return):
Move-ADDirectoryServerOperationMasterRole -Identity "DC02" `
  -OperationMasterRole PDCEmulator -Force
```

> **Gotcha:** **Never seize a role and then bring the old holder back online** — that creates a split-brain. Seize only when the old DC is gone for good, and metadata-clean it (`ntdsutil`) afterward.

### 5.7 Sites, subnets & replication **[A]**

In a multi-location company, **AD Sites** model your physical network topology so clients **authenticate against a *nearby* DC** and replication between locations is **scheduled and compressed over the WAN** instead of chattering constantly. You define **subnets** and associate them with **sites**; a client's IP tells AD which site it's in, and it's directed to that site's DCs.

- **Intra-site replication**: fast, uncompressed, change-notification driven (seconds-to-minutes).
- **Inter-site replication**: via **site links** with a **schedule and interval** (default 180 min), compressed to save bandwidth.

```powershell
# Map a subnet to a site so clients use local DCs:
New-ADReplicationSite   -Name "Branch-EU"
New-ADReplicationSubnet -Name "10.2.0.0/24" -Site "Branch-EU"

# Force replication / check health:
repadmin /replsummary           # quick health overview across all DCs
repadmin /syncall /AdeP         # push replicate from this DC to all partners
dcdiag /v                       # the big DC health test — run this when AD "feels weird"
```

### 5.8 Trusts — connecting domains/forests **[A]**

A **trust** lets users in one domain/forest be authenticated by another. Within a single forest, all domains trust each other automatically (two-way transitive). You create **explicit trusts** to share resources with a *separate* forest — e.g., after a company merger, or between a corporate forest and a locked-down "red" admin forest (ESAE/PAW model). Trusts can be **one-way or two-way**, **transitive or not**, and **external (single domain)** or **forest (whole forest)**. They're configured in *Active Directory Domains and Trusts*. Security-wise, treat a trust as a **hole in your boundary**: enable **selective authentication** and **SID filtering** so the trusted forest can't impersonate privileged SIDs.

### 5.9 Service accounts & gMSA — passwords humans never know **[I/A]**

Background services (IIS app pools, SQL Server, scheduled tasks, custom apps) need an **identity** to run under. The traditional approach — a normal domain user account with a static password typed into a config — is a security nightmare: the password rarely changes, lives in plaintext somewhere, and if stolen it's a permanent foothold. **group Managed Service Accounts (gMSA)** fix this: AD itself **generates and rotates a long, random password** (every 30 days by default), and only **authorized computers** can retrieve it at runtime — *no human ever knows or types it*. This is the modern best practice for any service identity.

```powershell
# ONE-TIME per forest: create the KDS root key (used to derive gMSA passwords).
# In production, wait 10 hours for replication; in a lab you can backdate it:
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))

# Create a gMSA and authorize which computers may use it:
New-ADServiceAccount -Name "svc_web" -DNSHostName "svc_web.contoso.local" `
  -PrincipalsAllowedToRetrieveManagedPassword "WEB01$","WEB02$" -Enabled $true

# On each server that will run the service, install it (note the trailing $ on the account name):
Install-ADServiceAccount -Identity "svc_web"
Test-ADServiceAccount -Identity "svc_web"     # should return True

# Then assign CONTOSO\svc_web$ as the identity of an IIS app pool, service, or scheduled task.
```

> **Security:** Prefer **gMSA** over standalone MSAs (single-host only) and over plain user service accounts. Give the account only the rights its service needs (least privilege), never Domain Admin, and never reuse one gMSA across unrelated services.

---

## 6. Managing AD — Objects at Scale, Delegation, and Group Policy in Depth

### 6.1 Creating and managing objects with PowerShell **[I]**

The GUI (**ADUC** — `dsa.msc`) is fine for one-off clicks, but real administration is scripted. The `ActiveDirectory` module is your toolkit:

```powershell
Import-Module ActiveDirectory   # installed with RSAT-AD-PowerShell / DC management tools

# Create an OU structure (the skeleton you target with GPO and delegation):
New-ADOrganizationalUnit -Name "Corp"  -Path "DC=contoso,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "OU=Corp,DC=contoso,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=Corp,DC=contoso,DC=local"

# Create a user with sensible defaults:
New-ADUser -Name "Jane Doe" -GivenName Jane -Surname Doe `
  -SamAccountName "jdoe" -UserPrincipalName "jdoe@contoso.local" `
  -Path "OU=Users,OU=Corp,DC=contoso,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Initial password") `
  -ChangePasswordAtLogon $true -Enabled $true

# Create groups (note GroupScope/GroupCategory — see the AGDLP note below):
New-ADGroup -Name "Sales-RW" -GroupScope DomainLocal -GroupCategory Security `
  -Path "OU=Groups,OU=Corp,DC=contoso,DC=local"
Add-ADGroupMember -Identity "Sales-RW" -Members jdoe
```

### 6.2 Bulk operations — the reason you learn PowerShell **[I]**

```powershell
# Bulk-create users from a CSV (Name,Sam,UPN,Dept columns):
Import-Csv .\newhires.csv | ForEach-Object {
    New-ADUser -Name $_.Name -SamAccountName $_.Sam -UserPrincipalName $_.UPN `
      -Department $_.Dept -Path "OU=Users,OU=Corp,DC=contoso,DC=local" `
      -AccountPassword (ConvertTo-SecureString "TempP@ss!23" -AsPlainText -Force) `
      -ChangePasswordAtLogon $true -Enabled $true
}

# Find & report: all users who haven't logged in for 90 days (great for cleanup/security):
$cut = (Get-Date).AddDays(-90)
Get-ADUser -Filter 'Enabled -eq $true' -Properties LastLogonDate |
  Where-Object { $_.LastLogonDate -lt $cut } |
  Select-Object Name, SamAccountName, LastLogonDate |
  Sort-Object LastLogonDate | Export-Csv .\stale-users.csv -NoTypeInformation

# Disable (don't delete) stale accounts; move to a "disabled" OU:
Get-ADUser -Filter 'Enabled -eq $true' -Properties LastLogonDate |
  Where-Object { $_.LastLogonDate -lt $cut } |
  ForEach-Object { Disable-ADAccount $_; Move-ADObject $_ -TargetPath "OU=Disabled,DC=contoso,DC=local" }
```

> **Best practice — AGDLP:** Microsoft's recommended group strategy is **A**ccounts → **G**lobal groups → **D**omain **L**ocal groups → **P**ermissions. Put user *Accounts* into *Global* groups by role (`Sales-Users`), put those Global groups into *Domain Local* groups that represent resource access (`Sales-RW`), and assign *Permissions* to the Domain Local group. This scales cleanly across domains and keeps permissioning sane.

### 6.3 Delegation — granting narrow admin rights **[I/A]**

**Delegation** lets you grant a person/team the ability to perform *specific* tasks on a *specific* OU (e.g., "the helpdesk can reset passwords for users in the `Branch-EU` OU") **without** making them Domain Admins. You do it with the **Delegation of Control Wizard** (right-click an OU in ADUC → *Delegate Control*) or by editing the OU's ACL. This is the AD-object equivalent of JEA, and a pillar of least privilege.

```powershell
# Programmatic delegation is verbose (raw ACE editing); the wizard is usually used.
# But you can grant, e.g., "reset password" on an OU via dsacls:
dsacls "OU=Users,OU=Corp,DC=contoso,DC=local" /I:S `
  /G "CONTOSO\Helpdesk:CA;Reset Password;user"
```

### 6.4 Group Policy (GPO) — what it is and why it's the backbone **[I/A]**

**Group Policy is the mechanism by which you push configuration and security settings to *every* domain-joined machine and user from a central place.** Want every workstation to lock after 10 minutes, every server to have a specific firewall rule, every user to get a mapped drive, password complexity enforced domain-wide, USB storage blocked, or a security baseline applied to 5,000 machines? That's Group Policy. It is, with AD itself, the reason companies run Windows Server.

A **GPO (Group Policy Object)** is a named bundle of settings (split into a **Computer Configuration** half and a **User Configuration** half). You **link** a GPO to a **site, domain, or OU**, and it applies to the machines/users beneath that link. Settings live in two big trees: **Policies** (enforced — users can't change them) and **Preferences** (set-once defaults users *can* change, used for drive maps, printers, registry, scheduled tasks).

You manage GPOs in the **Group Policy Management Console (GPMC** — `gpmc.msc`, part of RSAT).

### 6.5 How GPO applies: LSDOU, inheritance, and precedence **[I/A]**

Processing order is the famous acronym **LSDOU**: **L**ocal policy → **S**ite → **D**omain → **O**U (deepest OU last). **Later wins** on conflicts, so an OU-linked GPO overrides a domain-linked one. Modifiers:

| Mechanism | Effect |
|---|---|
| **Link order** | Lower number = higher priority *within the same container* (applied last). |
| **Enforced** (formerly "No Override") | Forces this GPO's settings to win and not be overridden by lower OUs. |
| **Block Inheritance** (on an OU) | Stops higher GPOs from flowing in — *except* Enforced ones, which punch through. |
| **Security/WMI filtering** | Apply a GPO only to members of a group, or only where a WMI query matches (e.g., only laptops). |
| **Loopback processing** | Apply *User* settings based on the *computer's* OU (kiosks, RDS hosts). |

```powershell
Import-Module GroupPolicy

# Create and link a GPO to an OU:
New-GPO -Name "Workstation Security Baseline" -Comment "Lock screen, firewall, defender"
New-GPLink -Name "Workstation Security Baseline" `
  -Target "OU=Computers,OU=Corp,DC=contoso,DC=local" -LinkEnabled Yes -Order 1

# Set a registry-backed policy value (e.g., inactivity lock = 600s):
Set-GPRegistryValue -Name "Workstation Security Baseline" `
  -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "InactivityTimeoutSecs" -Type DWord -Value 600

# Security-filter it to a specific group only:
Set-GPPermission -Name "Workstation Security Baseline" -TargetName "Laptops" `
  -TargetType Group -PermissionLevel GpoApply

# On a client, force a refresh and SEE what applied (the troubleshooting trio):
gpupdate /force
gpresult /h C:\report.html          # rich HTML report of resultant set of policy (RSoP)
gpresult /r                         # quick text summary in the console
```

### 6.6 Common, high-value policies **[I]**

| Goal | Where (Computer/User Config →) |
|---|---|
| Password length/complexity/lockout (domain-wide) | Computer → Policies → Windows Settings → Security → Account Policies (set at the **domain** root; see Fine-Grained Password Policies for exceptions) |
| Screen lock / inactivity | Security Options + InactivityTimeout |
| Windows Firewall rules at scale | Security → Windows Defender Firewall with Advanced Security |
| Drive mappings / printers | User → Preferences → Drive Maps / Printers |
| Block removable storage | Administrative Templates → System → Removable Storage Access |
| Deploy software / run scripts at logon | Software Settings; Scripts (Startup/Logon) |
| Apply a Microsoft Security Baseline | Import the SCT baseline GPOs (§13) |

> **⚡ Version note — Fine-Grained Password Policies (FGPP):** The classic rule "only one password policy per domain (at the domain root)" was relaxed long ago. Use **PSOs (Password Settings Objects)** to apply *different* password rules to specific groups (e.g., stricter for admins) within one domain. Manage via the **Active Directory Administrative Center** or `New-ADFineGrainedPasswordPolicy`.

> **Best practice:** Keep GPOs **small and purpose-named** ("Firewall – Servers", "Lockscreen – All"), back them up (`Backup-GPO -All`), document each one's intent, and test in a pilot OU before linking broadly. Avoid the monster "everything" GPO — it's impossible to troubleshoot.

---

## 7. DNS & DHCP on Windows Server

> These two roles are pure [Networking](NETWORKING_GUIDE.md) made operational on Windows. Read that guide for *what* DNS records and DHCP leases are; here we run the **servers** that provide them.

### 7.1 DNS — zones, records, and AD integration **[I]**

**DNS resolves names to addresses** and, for AD, publishes the **SRV records** that let clients find domain controllers (§5.3). On a DC, DNS is typically installed automatically and the AD zone is **AD-integrated** — stored *in* Active Directory, multi-master, and replicated with AD (no separate zone-transfer setup, and secure dynamic updates).

Key concepts:
- A **forward lookup zone** maps names→IPs (`contoso.local`); a **reverse lookup zone** maps IPs→names (used by logging, PTR records, some auth).
- **Record types** you'll create: **A/AAAA** (host→IPv4/IPv6), **CNAME** (alias), **MX** (mail), **SRV** (service location, the AD glue), **PTR** (reverse), **TXT** (SPF/verification).
- **Forwarders / Root hints**: how your DNS resolves names it isn't authoritative for. Point **conditional forwarders** at partner domains; set **forwarders** to your ISP or a public resolver for general internet resolution.

```powershell
Install-WindowsFeature DNS -IncludeManagementTools   # usually already present on a DC

# Create an AD-integrated forward zone and add records:
Add-DnsServerPrimaryZone -Name "app.contoso.local" -ReplicationScope "Domain"
Add-DnsServerResourceRecordA -ZoneName "contoso.local" -Name "web01" -IPv4Address "10.0.0.50"
Add-DnsServerResourceRecordCName -ZoneName "contoso.local" -Name "intranet" `
  -HostNameAlias "web01.contoso.local"

# A reverse zone + automatic PTR creation:
Add-DnsServerPrimaryZone -NetworkID "10.0.0.0/24" -ReplicationScope "Domain"

# Forward unknown queries to a public resolver:
Set-DnsServerForwarder -IPAddress 1.1.1.1, 9.9.9.9

# Troubleshoot resolution (use nslookup or the cmdlet):
Resolve-DnsName web01.contoso.local
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.contoso.local   # are DCs findable?
```

> **Security:** Enable **secure dynamic updates only** (AD-integrated zones do this) so rogue machines can't overwrite records. Consider **DNS scavenging** to purge stale records, and be aware of **DNSSEC** for signing zones. Restrict who can administer DNS; DNS admin is powerful (it borders on DC compromise).

### 7.2 DHCP — scopes, reservations, options, failover **[I]**

**DHCP hands out IP configuration** (address, subnet mask, gateway, DNS servers) to clients so you don't configure each by hand. The Windows DHCP role must be **authorized in AD** before it will serve leases (a safety feature preventing rogue DHCP servers).

- A **scope** is a range of addresses for one subnet, with a lease duration.
- **Exclusions** carve out addresses inside the scope you *don't* want handed out (e.g., for static servers).
- **Reservations** tie a specific MAC address to a specific IP — "static via DHCP," ideal for printers/servers you want centrally managed.
- **Options** deliver extra config: **003** (router/gateway), **006** (DNS servers), **015** (DNS domain name). Set them at server or scope level.
- **DHCP Failover** pairs two DHCP servers so leases survive a server outage — **Load Balance** mode (split 50/50) or **Hot Standby** mode (one primary, one backup).

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools

# Authorize the server in AD (mandatory) and add a security group:
Add-DhcpServerInDC -DnsName "dc01.contoso.local" -IPAddress 10.0.0.10
Add-DhcpServerSecurityGroup

# Create a scope, set options, add a reservation:
Add-DhcpServerv4Scope -Name "LAN" -StartRange 10.0.0.100 -EndRange 10.0.0.200 `
  -SubnetMask 255.255.255.0 -LeaseDuration 8.00:00:00
Set-DhcpServerv4OptionValue -ScopeId 10.0.0.0 -Router 10.0.0.1 `
  -DnsServer 10.0.0.10,10.0.0.11 -DnsDomain "contoso.local"
Add-DhcpServerv4ExclusionRange -ScopeId 10.0.0.0 -StartRange 10.0.0.100 -EndRange 10.0.0.110
Add-DhcpServerv4Reservation -ScopeId 10.0.0.0 -IPAddress 10.0.0.105 `
  -ClientId "AA-BB-CC-DD-EE-FF" -Description "Floor-2 printer"

# Configure FAILOVER between two DHCP servers (resilience):
Add-DhcpServerv4Failover -Name "DHCP-Failover" -PartnerServer "dc02.contoso.local" `
  -ScopeId 10.0.0.0 -LoadBalancePercent 50 -SharedSecret "ChangeMe!" -ServerRole Active
```

---

## 8. File Services — SMB, DFS, Quotas/FSRM, Storage Spaces, Dedup, VSS

### 8.1 SMB shares done right **[I]**

**SMB (Server Message Block)** is the Windows file-sharing protocol. §4.3 covered the share/NTFS permission model; here's the operational layer. **⚡ Version note:** modern Windows defaults to **SMB 3.1.1** with **AES-128/256 encryption** available per-share and **pre-auth integrity**; **SMB1 is removed/disabled** (it's insecure — never re-enable it). Server 2025 adds **SMB over QUIC** (file shares securely over the internet without a VPN) and **mandatory SMB signing** improvements.

```powershell
# A properly governed share with encryption and access-based enumeration:
New-SmbShare -Name "Finance" -Path "D:\Shares\Finance" `
  -FullAccess "Authenticated Users" `        # NTFS does the real gating (§4.3)
  -EncryptData $true `                        # encrypt SMB traffic for this share
  -FolderEnumerationMode AccessBased          # users only SEE folders they can access (ABE)

Get-SmbShare; Get-SmbShareAccess -Name "Finance"
Get-SmbConnection; Get-SmbOpenFile           # who is connected / what files are open

# Confirm SMB1 is gone (it must be):
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol
```

### 8.2 DFS — Namespaces & Replication **[I/A]**

Two distinct DFS technologies solve two problems:

- **DFS Namespaces (DFS-N)** gives users **one logical path** (`\\contoso.local\files\Finance`) that transparently points at shares on *any* server. Move the data to a new server, update the namespace target, and **users' mapped drives never change.** It decouples the path people use from the server the data lives on — huge for migrations and resilience.
- **DFS Replication (DFS-R)** keeps **multiple copies** of a folder in sync across servers/sites using efficient **remote differential compression** (only changed blocks travel). Combine with DFS-N to offer users a path that's served by whichever replica is nearest/alive.

```powershell
Install-WindowsFeature FS-DFS-Namespace, FS-DFS-Replication -IncludeManagementTools

# Domain-based namespace + a folder target:
New-DfsnRoot -TargetPath "\\dc01\files" -Type DomainV2 -Path "\\contoso.local\files"
New-DfsnFolder -Path "\\contoso.local\files\Finance" -TargetPath "\\fs01\Finance"
New-DfsnFolderTarget -Path "\\contoso.local\files\Finance" -TargetPath "\\fs02\Finance" # 2nd replica target

# Replicate Finance between fs01 and fs02:
New-DfsReplicationGroup -GroupName "RG-Finance"
New-DfsReplicatedFolder -GroupName "RG-Finance" -FolderName "Finance"
Add-DfsrMember -GroupName "RG-Finance" -ComputerName fs01, fs02
Add-DfsrConnection -GroupName "RG-Finance" -SourceComputerName fs01 -DestinationComputerName fs02
```

### 8.3 FSRM — quotas, file screens, reports **[I]**

**File Server Resource Manager** governs *what* and *how much* users can store: **quotas** (hard = block at limit, soft = warn only) on folders, **file screens** (block `.mp3`/`.exe` in a share), and **storage reports**. It's how you stop one user filling the volume and keep junk off the file server.

```powershell
Install-WindowsFeature FS-Resource-Manager -IncludeManagementTools
New-FsrmQuota -Path "D:\Shares\Finance" -Size 50GB -SoftLimit:$false `
  -Threshold (New-FsrmQuotaThreshold -Percentage 90)   # warn at 90%, hard-stop at 100%
# Block media files from a share:
New-FsrmFileGroup -Name "Media" -IncludePattern @("*.mp3","*.mp4","*.avi","*.mkv")
New-FsrmFileScreen -Path "D:\Shares\Finance" -IncludeGroup "Media" -Active   # active = block
```

### 8.4 Storage Spaces & deduplication **[I/A]**

**Storage Spaces** pools physical disks into a **storage pool**, from which you carve **virtual disks** with software resiliency (mirror/parity) — software RAID, basically, but flexible and tiering-aware. (Full coverage in §11.) **Data Deduplication** is a file-server feature that finds duplicate chunks across files and stores them once — typically **50–90% savings** on file shares, VHD libraries, and backups, at the cost of some CPU/RAM.

```powershell
Install-WindowsFeature FS-Data-Deduplication -IncludeManagementTools
Enable-DedupVolume -Volume "D:" -UsageType Default      # or HyperV / Backup usage types
Start-DedupJob -Volume "D:" -Type Optimization
Get-DedupStatus -Volume "D:"                            # see saved space
```

### 8.5 Shadow Copies / VSS — "Previous Versions" **[I]**

**Volume Shadow Copy Service (VSS)** takes point-in-time snapshots of a volume so users can self-service restore an earlier version of a file via the **Previous Versions** tab — recovering accidental deletes/overwrites *without* calling IT or hitting backups. It is **not a backup** (snapshots live on the same volume), but it's a fantastic first line of recovery.

```powershell
# Schedule shadow copies on D: (defaults: 7:00 & 12:00 on weekdays). vssadmin is the CLI:
vssadmin add shadowstorage /for=D: /on=D: /maxsize=10%
vssadmin create shadow /for=D:
vssadmin list shadows
# Users restore via Explorer: right-click file/folder -> "Restore previous versions".
```

### 8.6 Print services **[I]**

The other half of "file **and** print services." The **Print and Document Services** role turns a server into a **print server**: you install printer drivers and share printers centrally, and clients connect to `\\printsrv\PrinterName` (or get printers pushed via GPO). Centralizing printing means drivers, defaults, and permissions are managed in one place, and you can audit/queue-manage jobs. **⚡ Version note:** because of historic print-spooler vulnerabilities (the "PrintNightmare" class of bugs), modern guidance is to **disable the Print Spooler on any server that isn't deliberately a print server** (especially DCs), patch print servers promptly, and restrict who can install drivers via the **Point and Print** hardening policies.

```powershell
Install-WindowsFeature Print-Services -IncludeManagementTools

# Add a TCP/IP printer port, install a driver, create and share a printer:
Add-PrinterPort   -Name "IP_10.0.0.60" -PrinterHostAddress "10.0.0.60"
Add-PrinterDriver -Name "Generic / Text Only"          # use the real vendor driver in practice
Add-Printer -Name "HR-Laser" -DriverName "Generic / Text Only" -PortName "IP_10.0.0.60" -Shared -ShareName "HR-Laser"
Get-Printer; Get-PrintJob -PrinterName "HR-Laser"      # manage the queue

# SECURITY: on servers that should NOT print, disable the spooler entirely (e.g., DCs):
Stop-Service Spooler; Set-Service Spooler -StartupType Disabled
```

> **Security:** The Print Spooler runs as SYSTEM and has been a repeated remote-code-execution target. Treat a print server as a hardened, patched, single-purpose box; **disable the spooler everywhere it isn't needed**, and lock down **Point and Print** so non-admins can't trigger driver installs.

---

## 9. Web & App Hosting — IIS in Depth

> Put a hardened reverse proxy in front of app servers where it helps — see the [Nginx](NGINX_GUIDE.md) guide for that pattern. IIS itself is a capable, production web server, and the default home for **.NET** apps. The TLS/cert fundamentals come from the [Networking](NETWORKING_GUIDE.md) guide.

### 9.1 What IIS is and its core objects **[I]**

**Internet Information Services (IIS)** is Windows's built-in web server. Four objects structure everything:

| Object | What it is |
|---|---|
| **Site** | A website: a name, one or more **bindings**, and a content root. |
| **Binding** | The (protocol, IP, port, hostname) tuple that routes requests to a site — e.g., `https`, `*`, `443`, `www.contoso.com`. |
| **Application** | A path within a site that's its own app (own settings), e.g., `/api`. |
| **Application Pool** | The **worker process(es)** (`w3wp.exe`) that run a site's code, with their own **identity**, .NET/runtime version, and recycling settings. Isolation between apps = isolation between pools. |

The big idea: **app pools isolate sites from each other.** A crash or compromise in one pool's worker process doesn't take down sites in other pools, and each pool runs under a **distinct, low-privilege identity** (`ApplicationPoolIdentity` by default — a virtual account scoped to that pool).

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools   # the IIS role
Import-Module WebAdministration                              # or the newer IISAdministration module

# Create an app pool and a site bound to it:
New-WebAppPool -Name "ContosoAppPool"
Set-ItemProperty IIS:\AppPools\ContosoAppPool -Name managedRuntimeVersion -Value ""  # "" = No Managed Code (for .NET Core/Kestrel behind IIS)
New-Website -Name "Contoso" -PhysicalPath "C:\inetpub\contoso" `
  -ApplicationPool "ContosoAppPool" -Port 80 -HostHeader "www.contoso.com"

# Add a sub-application:
New-WebApplication -Site "Contoso" -Name "api" -PhysicalPath "C:\inetpub\contoso-api" `
  -ApplicationPool "ContosoAppPool"
```

### 9.2 App pool identity, recycling, and health **[I]**

```powershell
# Recycle daily at 03:00 and after 1500 MB private memory (tune to your app):
Set-ItemProperty IIS:\AppPools\ContosoAppPool -Name recycling.periodicRestart.time -Value "00:00:00"
Set-ItemProperty IIS:\AppPools\ContosoAppPool -Name recycling.periodicRestart.schedule `
  -Value @{value="03:00:00"}
Set-ItemProperty IIS:\AppPools\ContosoAppPool -Name recycling.periodicRestart.privateMemory -Value 1536000

# Run the pool under a gMSA (best for apps needing domain identity — auto-rotated password):
Set-ItemProperty IIS:\AppPools\ContosoAppPool -Name processModel.identityType -Value SpecificUser
# (gMSA setup is in §5/§13; assign it here as the identity.)
```

> **Security:** Keep pools on **ApplicationPoolIdentity** or a **gMSA**, never a domain admin. One app per pool where feasible. Remove unneeded IIS modules/handlers. Set `requestFiltering` limits. Don't run app code as the pool that also has filesystem write to its own binaries.

### 9.3 TLS certificates and HTTPS bindings **[I/A]**

```powershell
# Bind an existing cert (by thumbprint) to the site on 443:
New-WebBinding -Name "Contoso" -Protocol https -Port 443 -HostHeader "www.contoso.com" -SslFlags 1
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like "*contoso.com*"
New-Item -Path "IIS:\SslBindings\!443!www.contoso.com" -Value $cert -SSLFlags 1  # SNI

# Request an internal cert from your AD Certificate Services CA (enterprise PKI):
Get-Certificate -Template WebServer -DnsName "www.contoso.com" -CertStoreLocation Cert:\LocalMachine\My
```

> **⚡ Version note:** Harden TLS via the OS Schannel settings / a security baseline (disable TLS 1.0/1.1, weak ciphers), enable **HSTS** (native `<hsts>` element in IIS 10+), and prefer **TLS 1.3**. For public sites you'll often terminate TLS at an Nginx/HAProxy/Azure front door instead — see [Nginx](NGINX_GUIDE.md).

### 9.4 ARR / reverse proxy vs Nginx in front **[I/A]**

IIS can be a **reverse proxy** using **Application Request Routing (ARR)** + **URL Rewrite** — load-balancing and forwarding to backend app servers (including Kestrel/.NET, Node, or Linux upstreams). It's legitimate, but many shops prefer **Nginx** in front of IIS/app servers for its simpler config, performance, and cross-platform consistency (see the [Nginx](NGINX_GUIDE.md) guide for that exact pattern). Rule of thumb: **ARR if you're all-IIS and want one stack to patch; Nginx if you're heterogeneous or want a battle-tested edge.**

### 9.5 Hosting .NET apps **[I]**

Modern **ASP.NET Core** apps run on the **Kestrel** server inside their own process; IIS sits in front as a reverse proxy via the **ASP.NET Core Module (ANCM)**, set the pool to **"No Managed Code."** Classic **ASP.NET (Framework)** apps run *in* the worker process with the matching CLR version. Either way: deploy to a folder, point the site/app at it, set the pool identity, and ensure the runtime/hosting bundle is installed. For database-backed apps, see [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md).

---

## 10. Hyper-V Virtualization

### 10.1 Why virtualize, and what Hyper-V is **[I]**

**Hyper-V is Windows Server's built-in type-1 (bare-metal) hypervisor.** Virtualization lets one physical server run many isolated **virtual machines (VMs)**, each with its own OS — consolidating hardware, isolating workloads, enabling snapshots/rollback, and making servers movable between hosts. It's the foundation of on-prem private cloud and a prerequisite for understanding the cloud. (For *containers* rather than full VMs, see the [Docker](DOCKER_GUIDE.md) guide — Windows containers can even use Hyper-V isolation.)

```powershell
Install-WindowsFeature Hyper-V -IncludeManagementTools -Restart   # needs CPU virtualization (VT-x/AMD-V)
```

### 10.2 Virtual switches — how VMs reach the network **[I]**

A **virtual switch** connects VMs to networking. Three types:

| Switch type | VMs can reach… |
|---|---|
| **External** | The physical network (and each other) — bound to a physical NIC. The normal choice. |
| **Internal** | Each other **and** the host only — no physical network. |
| **Private** | Each other only — fully isolated lab. |

```powershell
New-VMSwitch -Name "External" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

### 10.3 Creating and managing VMs **[I/A]**

```powershell
# Create a Gen-2 VM (UEFI, Secure Boot — use Gen2 for modern OSes):
New-VM -Name "WEB01" -Generation 2 -MemoryStartupBytes 4GB `
  -NewVHDPath "D:\VMs\WEB01\WEB01.vhdx" -NewVHDSizeBytes 80GB -SwitchName "External"

Set-VM -Name "WEB01" -DynamicMemory -MemoryMinimumBytes 2GB -MemoryMaximumBytes 8GB `
  -ProcessorCount 4 -AutomaticStartAction Start -AutomaticStopAction Save
Set-VMDvdDrive -VMName "WEB01" -Path "D:\ISO\WinServer2025.iso"   # attach install media
Start-VM -Name "WEB01"

Get-VM; Get-VMNetworkAdapter -VMName "WEB01"
```

### 10.4 Checkpoints (snapshots) **[I]**

A **checkpoint** captures a VM's state so you can roll back — invaluable before a risky change. **Use "Production" checkpoints** (VSS-consistent, application-aware) not "Standard" for anything stateful, and **never rely on checkpoints as backups** or run a domain controller from a reverted checkpoint without USN-rollback awareness (it can corrupt AD replication).

```powershell
Checkpoint-VM -Name "WEB01" -SnapshotName "Before patch"
Get-VMSnapshot -VMName "WEB01"
Restore-VMSnapshot -Name "Before patch" -VMName "WEB01" -Confirm:$false
Remove-VMSnapshot -VMName "WEB01" -Name "Before patch"   # merges the differencing disk — do this promptly
```

> **Gotcha:** Checkpoints grow **differencing disks** that consume space and *slow the VM* the longer they exist. Delete them after you've confirmed a change is good. They are **not** a backup strategy.

### 10.5 Live Migration & clustering **[A]**

**Live Migration** moves a *running* VM from one Hyper-V host to another with **no downtime**, by copying memory pages while the VM runs, then a final cutover. It needs shared/SMB storage or Storage Migration and either a cluster or configured constrained-delegation/Kerberos. Combined with **Failover Clustering**, VMs **automatically restart on a surviving host** if one host dies — the basis of HA virtualization. **Storage Spaces Direct (S2D, Datacenter edition)** turns a cluster's local disks into shared, resilient storage for this.

```powershell
Enable-VMMigration
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos -UseAnyNetworkForMigration $true
Move-VM -Name "WEB01" -DestinationHost "HV02" -IncludeStorage -DestinationStoragePath "D:\VMs\WEB01"
```

---

## 11. Storage & Disks — Partitioning, Storage Spaces, iSCSI, ReFS vs NTFS

### 11.1 Disk basics: MBR vs GPT, initialize/partition/format **[I]**

A new disk must be **initialized** (with a partition style), **partitioned**, and **formatted** before use. **GPT (GUID Partition Table)** is the modern standard — supports >2 TB disks and many partitions, required for UEFI boot. **MBR** is legacy (≤2 TB, ≤4 primary partitions). **Always choose GPT** for new data disks.

```powershell
Get-Disk                                    # list disks and partition styles
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter E |
  Format-Volume -FileSystem NTFS -NewFileSystemLabel "Data" -Confirm:$false
Get-Volume                                  # see all volumes, free space, health
# Legacy interactive tool: diskpart  (list disk / select disk N / clean / create partition primary ...)
```

### 11.2 Storage Spaces & resiliency **[I/A]**

**Storage Spaces** abstracts physical disks into a **pool**, then lets you create **virtual disks (spaces)** with a chosen **resiliency**:

| Resiliency | Protects against | Cost | Use |
|---|---|---|---|
| **Simple** | nothing (striping for speed) | 0 redundancy | scratch/temp only |
| **Mirror (2-way / 3-way)** | 1 / 2 disk failures | 50% / 67% overhead | general, performance |
| **Parity / dual parity** | 1 / 2 disk failures | space-efficient, slower writes | archival, backups |

```powershell
$disks = Get-PhysicalDisk -CanPool $true
New-StoragePool -FriendlyName "Pool1" -StorageSubsystemFriendlyName (Get-StorageSubsystem).FriendlyName `
  -PhysicalDisks $disks
New-VirtualDisk -StoragePoolFriendlyName "Pool1" -FriendlyName "DataVD" `
  -ResiliencySettingName Mirror -UseMaximumSize -ProvisioningType Thin
Get-VirtualDisk "DataVD" | Get-Disk | Initialize-Disk -PassThru |
  New-Partition -UseMaximumSize -AssignDriveLetter |
  Format-Volume -FileSystem ReFS -NewFileSystemLabel "Data" -Confirm:$false
```

> **⚡ Version note:** **Storage Spaces Direct (S2D)** — Datacenter edition — pools the *local* disks of a cluster into shared resilient storage for Hyper-V/SQL, no SAN required. It underpins Azure Stack HCI / Windows Server hyper-converged infrastructure.

### 11.3 ReFS vs NTFS **[I/A]**

| | **NTFS** | **ReFS (Resilient File System)** |
|---|---|---|
| Maturity / compatibility | Universal default, supports everything (boot, dedup, EFS, quotas, compression) | Newer; *cannot* boot Windows; no EFS/short-names |
| Integrity | Journaling | **Integrity streams + checksums + auto-repair** (with mirror/parity) |
| Strengths | The safe everyday choice | **Huge volumes, Hyper-V/SQL data, fast VHDX ops (block cloning), bit-rot detection** |
| Use it for | OS disks, general file shares | Big data volumes, virtualization/backup storage on resilient pools |

**Rule of thumb:** **NTFS** for the OS and general shares; **ReFS** for large data/virtualization volumes that sit on mirror/parity Storage Spaces (so its self-healing has redundancy to repair from).

### 11.4 iSCSI — block storage over the network **[I/A]**

**iSCSI** presents **block storage** (a raw "disk") to a server *over the network*, as if it were a local drive — the cheap, Ethernet-based alternative to Fibre Channel SAN. The storage box is the **target**; the server consuming it is the **initiator**. Windows Server can be both. Used for shared cluster storage, big data volumes, and lab SANs.

```powershell
# Make THIS server an iSCSI TARGET (share a virtual disk to other servers):
Install-WindowsFeature FS-iSCSITarget-Server -IncludeManagementTools
New-IscsiVirtualDisk  -Path "D:\iSCSI\lun0.vhdx" -SizeBytes 100GB
New-IscsiServerTarget -TargetName "cluster-target" -InitiatorIds "IQN:iqn.1991-05.com.microsoft:srv01"
Add-IscsiVirtualDiskTargetMapping -TargetName "cluster-target" -Path "D:\iSCSI\lun0.vhdx"

# On the CONSUMING server (INITIATOR), connect to the target:
Start-Service msiscsi; Set-Service msiscsi -StartupType Automatic
New-IscsiTargetPortal -TargetPortalAddress "10.0.0.20"
Get-IscsiTarget | Connect-IscsiTarget -IsPersistent $true
# Then Initialize-Disk / New-Partition / Format-Volume the new disk as in §11.1.
```

---

## 12. Networking & Firewall on Windows Server

> The [Networking](NETWORKING_GUIDE.md) guide explains IP, subnets, TCP/UDP, and TLS; here we configure those on the Windows host and lock it down with the firewall.

### 12.1 Windows Defender Firewall with Advanced Security **[I]**

Windows has a **stateful, profile-aware host firewall** that is **on by default** and should *stay* on (turning it off is a common, dangerous mistake). It has three **profiles** — **Domain** (connected to your AD network), **Private** (trusted home/lab), **Public** (untrusted) — each with its own ruleset, and a default of **block inbound / allow outbound**. You add **inbound rules** to permit the services this server offers (RDP, SMB, the app's port) and, in high-security environments, **outbound rules** too. At scale you push these via **Group Policy** (§6) so every server gets identical rules.

```powershell
# State and profiles:
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True   # keep it ON

# Allow a specific inbound app port from a specific subnet only (least exposure):
New-NetFirewallRule -DisplayName "App HTTPS 8443" -Direction Inbound -Action Allow `
  -Protocol TCP -LocalPort 8443 -RemoteAddress 10.0.0.0/24 -Profile Domain

# Enable a built-in rule group instead of crafting rules by hand:
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Get-NetFirewallRule -Enabled True -Direction Inbound | Select-Object DisplayName, Profile, Action
```

> **Security best practice:** Default-deny inbound; open only the exact ports needed; scope rules to **specific remote subnets**; manage rules centrally via GPO so they're consistent and auditable. **Never** "disable the firewall to test" and forget to re-enable it.

### 12.2 NIC teaming / SET **[I]**

**NIC Teaming** bonds multiple physical network adapters into one logical NIC for **fault tolerance** (survive a cable/port failure) and **bandwidth aggregation**. **⚡ Version note:** the classic **LBFO** teaming is legacy; on Hyper-V hosts the modern approach is **Switch Embedded Teaming (SET)** integrated with the virtual switch.

```powershell
# Classic LBFO team (non-Hyper-V hosts):
New-NetLbfoTeam -Name "Team1" -TeamMembers "Ethernet","Ethernet 2" `
  -TeamingMode SwitchIndependent -LoadBalancingAlgorithm Dynamic
# Modern SET (created with the vSwitch on Hyper-V hosts):
New-VMSwitch -Name "SET-Switch" -NetAdapterName "Ethernet","Ethernet 2" -EnableEmbeddedTeaming $true
```

### 12.3 Routing & RRAS **[I]**

Windows Server can route between networks, run a VPN, or do NAT via the **Routing and Remote Access Service (RRAS)** role. In modern environments dedicated appliances usually handle this, but RRAS still serves for site VPNs, simple routing, and lab gateways. Add it with `Install-WindowsFeature RemoteAccess -IncludeManagementTools` and configure via the **Routing and Remote Access** console or `Add-VpnConnection`/`netsh routing` equivalents.

---

## 13. Security & Hardening

> This is the section that separates a sysadmin from a *trusted* one. Windows Server in production is a target; everything here reduces the attacker's options. Pair with the host firewall (§12), least privilege (§4.5), JEA (§3.5), and the operations practices in §17.

### 13.1 The threat model in one paragraph **[A]**

Modern attacks on Windows estates follow a pattern: **phish a user → run code on their workstation → steal credentials/tokens from memory → move laterally → harvest a privileged credential → reach a Domain Controller → game over (the whole forest).** Nearly every hardening control below exists to break one link in that chain: stop code running (ASR), stop credential theft (Credential Guard, Protected Users, LAPS), stop lateral movement (firewall, tiering), and detect/recover when something gets through (auditing, baselines, backup).

### 13.2 Microsoft Defender Antivirus & attack surface reduction (ASR) **[A]**

**Microsoft Defender Antivirus** is built in, always-on, and genuinely good — don't replace it lightly. **ASR rules** are a powerful, free hardening layer that blocks common malware *techniques* (Office spawning child processes, credential theft from LSASS, script-launched executables, etc.). Enable them in **audit** first, watch for false positives, then **block**.

```powershell
Get-MpComputerStatus | Select-Object AMRunningMode, RealTimeProtectionEnabled, AntivirusSignatureVersion
Update-MpSignature                              # pull the latest definitions
Start-MpScan -ScanType QuickScan

# Block credential theft from LSASS (an extremely high-value ASR rule). GUID is Microsoft's.
Add-MpPreference -AttackSurfaceReductionRules_Ids 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 `
  -AttackSurfaceReductionRules_Actions Enabled     # use 'AuditMode' first in production
```

> **Enterprise note:** Defender for Endpoint (the EDR/"ATP" cloud service) adds detection, threat hunting, and automated response across the fleet, managed centrally (often via Intune/Defender portal or Azure Arc). Push ASR and Defender settings to all servers via **GPO** or Intune.

### 13.3 Credential Guard & Protected Users — stopping credential theft **[A]**

The classic Windows attack is dumping cached credentials/hashes from the **LSASS** process to "pass the hash" laterally. Two big defenses:

- **Credential Guard** uses **virtualization-based security (VBS)** to isolate secrets (NTLM hashes, Kerberos TGTs) in a hardware-protected enclave that even kernel-level malware can't read. **⚡ Version note:** on modern Windows it can be enabled by default where hardware allows; verify and enforce it via GPO/registry.
- The **Protected Users** group hardens its members (put your *admins* in it): it disables NTLM, weak Kerberos ciphers, and credential caching for those accounts, drastically shrinking what an attacker can steal.

```powershell
# Add admin accounts to Protected Users (do NOT add service/computer accounts blindly):
Add-ADGroupMember -Identity "Protected Users" -Members "admin.jdoe"

# Verify Credential Guard is running:
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select-Object SecurityServicesRunning   # value 1 = Credential Guard running
```

### 13.4 LAPS — fixing the shared local-admin-password problem **[A]**

If every machine has the **same** local Administrator password, one compromised box compromises them all (lateral movement heaven). **Windows LAPS (Local Administrator Password Solution)** — now **built into Windows** (no add-on) — automatically sets a **unique, random** local admin password per machine, rotates it on a schedule, and stores it securely in **AD or Entra ID**, readable only by authorized admins. This single control shuts down one of the most common lateral-movement paths.

```powershell
# Windows LAPS (built-in). Configure primarily via GPO/Intune; PowerShell module helps verify:
Get-Command -Module LAPS
# Retrieve the current managed password for a computer (authorized admins only):
Get-LapsADPassword -Identity "WS-FINANCE-07" -AsPlainText
# Force a rotation now:
Reset-LapsPassword -Identity "WS-FINANCE-07"
```

### 13.5 Security baselines — Microsoft Security Compliance Toolkit (SCT) **[A]**

You should **not** hand-craft hundreds of security settings. Microsoft publishes **Security Baselines** — curated, tested GPO packs of recommended settings for each Windows Server/Client version — in the **Security Compliance Toolkit (SCT)**, along with the **Policy Analyzer** (compare your settings to the baseline) and **LGPO.exe** (apply baselines to standalone machines). Import the baseline GPOs, review/adjust for your environment, and link them. This gives you a defensible, industry-aligned starting posture in an afternoon instead of months. (CIS Benchmarks are a popular alternative/complement.)

```text
Workflow:
 1. Download the SCT + the Windows Server 2025 baseline (offline package).
 2. Use Policy Analyzer to diff the baseline vs your current GPOs (find gaps/conflicts).
 3. Import the baseline GPOs into GPMC; tweak any settings that break your apps (test in a pilot OU!).
 4. Link to a pilot OU -> validate -> roll out to all servers.
 5. Re-run Policy Analyzer periodically to catch drift.
```

### 13.6 BitLocker — encryption at rest **[A]**

**BitLocker** encrypts whole volumes so a stolen disk/server reveals nothing. On servers it protects data drives (and OS drives) using the **TPM** chip (and optionally a PIN/startup key). Recovery keys must be **escrowed in AD** so you can recover an encrypted volume.

```powershell
Install-WindowsFeature BitLocker -IncludeManagementTools -Restart
Enable-BitLocker -MountPoint "D:" -EncryptionMethod XtsAes256 -UsedSpaceOnly `
  -TpmProtector
Add-BitLockerKeyProtector -MountPoint "D:" -RecoveryPasswordProtector   # escrow this to AD via GPO
Get-BitLockerVolume | Select-Object MountPoint, VolumeStatus, EncryptionPercentage
```

### 13.7 Patch management — Patch Tuesday, WSUS, and modern alternatives **[A]**

Unpatched servers are the most common breach vector. Microsoft ships security updates on **"Patch Tuesday"** (the second Tuesday each month). At scale you don't let each server pull from the internet ad hoc — you **control and stage** updates:

- **WSUS (Windows Server Update Services)** is the classic on-prem role: it downloads updates once, you **approve** them per group of machines, and clients pull from WSUS. Gives you control and bandwidth savings, but is dated.
- **⚡ Version note:** Microsoft is steering customers toward **Windows Autopatch / Microsoft Intune / Azure Update Manager (via Azure Arc)** for modern, cloud-driven patch orchestration of even on-prem servers. WSUS still works and is widely deployed, but is on a deprecation trajectory — plan accordingly.

```powershell
Install-WindowsFeature UpdateServices -IncludeManagementTools   # WSUS role
# Approve updates to a target group, decline superseded ones, etc., via the WSUS console or the
# UpdateServices PowerShell module. Clients are pointed at WSUS via GPO:
#   Computer Config -> Admin Templates -> Windows Components -> Windows Update -> Specify intranet update service location.

# On a single server, check/install pending updates (built-in or PSWindowsUpdate module):
Get-WindowsUpdate -AcceptAll -Install -AutoReboot   # (PSWindowsUpdate)
```

> **Best practice:** Patch in **rings** — a small pilot ring first, then broad — and *always after a recent backup*. Maintain a documented **maintenance window** and rollback plan. Never disable updates "to keep things stable"; stale = breached.

### 13.8 Auditing — knowing what happened **[A]**

You can't investigate or prove compliance without **audit logs**. The **Advanced Audit Policy** (set via GPO) records security-relevant events: logons (4624/4625), account/group changes, privilege use, object access, process creation (4688, with command-line auditing), and Directory Service changes. Forward these to a central collector/SIEM (§14).

```powershell
# Inspect/set audit subcategories (auditpol is the tool; production = set via GPO baseline):
auditpol /get /category:*
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
# Enable command-line capture in 4688 events (huge for hunting):
#   GPO: Admin Templates -> System -> Audit Process Creation -> Include command line.
```

### 13.9 Hardening checklist **[A]**

| Control | Why |
|---|---|
| **Server Core** for infra roles | Smaller attack surface, fewer patches |
| Keep **firewall on**, default-deny inbound | Limit reachable services |
| **No web browsing / email** from servers | Removes the #1 initial-access path |
| **Separate admin accounts**, tiered (§17) | Contain credential theft |
| **LAPS** for local admin | Kill same-password lateral movement |
| **Protected Users** + **Credential Guard** | Stop hash/ticket theft |
| **ASR rules** + Defender + EDR | Block techniques, detect intrusions |
| **Security baselines** (SCT/CIS) applied via GPO | Defensible posture, no guesswork |
| **BitLocker** on data/OS drives | Encryption at rest |
| **Patch** monthly in rings | Close known holes |
| **Auditing** + central log forwarding | Detect & investigate |
| **Backups** tested + offline/immutable copy | Survive ransomware (§15) |
| Disable **SMB1**, **NTLM** where possible, legacy ciphers | Remove weak protocols |

---

## 14. Monitoring & Logging

### 14.1 Event Viewer & the Windows event logs **[I]**

Windows logs everything to structured **event logs**. The classic three are **System** (drivers, services, hardware), **Application** (app/role events), and **Security** (audit events — logons, policy changes; §13.8). Plus hundreds of per-component **Applications and Services Logs**. Each event has a **source**, **Event ID**, level (Information/Warning/Error/Critical), and detail. Learning to read these is core troubleshooting.

```powershell
# Modern cmdlet (PowerShell-friendly objects). Get-EventLog is the old, classic-logs-only one.
Get-WinEvent -LogName System -MaxEvents 50 | Where-Object Level -le 3   # warnings+ (1-3)
Get-WinEvent -FilterHashtable @{ LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-24) } |
  Select-Object TimeCreated, @{n='User';e={$_.Properties[5].Value}}, @{n='Source';e={$_.Properties[19].Value}}
# ^ 4625 = failed logon. Spikes = brute-force / spray attempts.

# Across many servers at once:
Invoke-Command -ComputerName $servers -ScriptBlock {
  Get-WinEvent -FilterHashtable @{ LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-1) }
} | Sort-Object TimeCreated -Descending | Select-Object PSComputerName, TimeCreated, Id, Message -First 50
```

### 14.2 Event forwarding (WEF) — centralizing logs **[I/A]**

Logging on each box is useless if an attacker wipes it. **Windows Event Forwarding (WEF)** pushes events from many "source" machines to a central **collector** server (subscription-based, agentless, built-in). The collector becomes your single place to query — and feeds a **SIEM** (Sentinel, Splunk, Elastic). Configure with `wecutil`/`winrm qc -q` and a subscription, then point your detection at the collector.

```text
# On the collector:  wecutil qc            (enable the collector service)
# On sources:        winrm qc              (enable WinRM)  + add collector to "Event Log Readers"
# Define a subscription (which events, from which machines) in Event Viewer -> Subscriptions,
#   or via a wecutil XML config. Forward Security 4624/4625/4688, AppLocker, PowerShell logs, etc.
```

### 14.3 Performance Monitor & resource baselining **[I/A]**

**Performance Monitor (`perfmon`)** graphs and logs **performance counters** — CPU, memory, disk queue length, network, and role-specific counters (AD, IIS, SQL). The pro move is to capture a **Data Collector Set** as a *baseline* when the server is healthy, so when it's slow later you can compare. Watch the classics:

| Counter | Healthy-ish guideline | Means |
|---|---|---|
| `\Processor(_Total)\% Processor Time` | sustained < 80% | CPU pressure |
| `\Memory\Available MBytes` | not near zero | RAM exhaustion |
| `\Memory\Pages/sec` | low | hard paging (RAM starvation) |
| `\PhysicalDisk(*)\Avg. Disk sec/Read` (latency) | < 20 ms | slow storage |
| `\PhysicalDisk(*)\Current Disk Queue Length` | low | disk bottleneck |
| `\Network Interface(*)\Output Queue Length` | ~0 | network saturation |

```powershell
# Quick live snapshot without the GUI:
Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes',
            '\PhysicalDisk(_Total)\Avg. Disk sec/Read' -SampleInterval 2 -MaxSamples 5
# Create a logging Data Collector Set with logman (runs headless on Core):
logman create counter Baseline -c "\Processor(_Total)\% Processor Time" "\Memory\Available MBytes" `
  -si 15 -o C:\PerfLogs\Baseline -f bincirc -max 200
logman start Baseline
```

### 14.4 Windows Admin Center as a monitoring pane **[I]**

**Windows Admin Center** gives live CPU/memory/disk/network charts, event log browsing, running processes, services, and role dashboards across all your servers from one browser — the friendliest day-to-day monitoring for small/medium estates without standing up a full monitoring stack. For larger estates, integrate with **Azure Monitor (via Azure Arc)** or a third-party tool (PRTG, Zabbix, Datadog) for alerting, dashboards, and history.

### 14.5 What to actually watch **[I/A]**

- **Health:** disk free space (alert < 15%), service up/down for role services, failed scheduled tasks, hardware (via vendor agents/iLO/iDRAC), reboot-pending state.
- **Security:** failed-logon spikes (4625), new members of privileged groups (4728/4732/4756), account lockouts (4740), service/scheduled-task creation, AppLocker/ASR blocks.
- **Capacity:** CPU/RAM/disk-latency trends for planning; AD replication health (`repadmin /replsummary`); backup success/failure.

---

## 15. Backup & Disaster Recovery

> The job nobody thanks you for — until the day it saves the company. Ransomware has made **tested, isolated backups** the single most important operational control. RTO/RPO and AD recovery are the parts beginners most often get wrong.

### 15.1 RTO, RPO, and what DR actually means **[I]**

Two numbers drive every backup decision:

- **RPO (Recovery Point Objective)** — *how much data can you afford to lose?* It dictates **backup frequency**. RPO of 1 hour = back up at least hourly.
- **RTO (Recovery Time Objective)** — *how fast must you be back up?* It dictates your **restore strategy** (warm standby vs restore-from-tape).

**Disaster Recovery** is the plan/runbook to meet those targets after a serious failure (site loss, ransomware, hardware death). A backup you've never *test-restored* is not a backup — it's a hope. The professional rule is **3-2-1**: **3** copies, on **2** media types, with **1** offsite — and a modern addition: **1 immutable/air-gapped** copy ransomware can't encrypt.

### 15.2 Windows Server Backup **[I]**

The built-in backup feature handles file/folder, volume, **system state** (the OS+roles config, registry, AD database on a DC), and **bare-metal recovery (BMR)** images. It's adequate for single servers; enterprises layer a dedicated product (Veeam, Commvault, Azure Backup) for orchestration, dedup, and offsite.

```powershell
Install-WindowsFeature Windows-Server-Backup

# One-off: back up the whole system (BMR + system state) to a target volume:
wbadmin start backup -backupTarget:E: -include:C: -allCritical -systemState -quiet

# Schedule a daily backup (the cmdlet way builds a policy):
$pol = New-WBPolicy
Add-WBSystemState $pol
Add-WBBareMetalRecovery $pol
Add-WBVolume -Policy $pol -Volume (Get-WBVolume -AllVolumes | Where-Object MountPath -eq 'C:')
Set-WBSchedule -Policy $pol -Schedule 02:00
Set-WBBackupTarget -Policy $pol -Target (New-WBBackupTarget -VolumePath 'E:')
Set-WBPolicy -Policy $pol
Get-WBJob; Get-WBSummary                       # status & history
```

### 15.3 System state & the special case of Domain Controllers **[I/A]**

A DC's **system state** includes the **AD database (NTDS.dit)** and **SYSVOL**. Restoring a DC is *not* like restoring a file server, because of **replication** — and getting it wrong can resurrect deleted objects forest-wide or cause **USN rollback** (a replication-poisoning condition). Two restore modes you must understand:

- **Non-authoritative restore** (the normal one): restore the DC's system state, boot it, and let it **replicate the current state** from healthy DCs to catch up. Use when a DC died but the directory is otherwise fine.
- **Authoritative restore** (`ntdsutil`): restore *and mark specific objects/subtrees as authoritative* so they **overwrite** what's on other DCs — used to **bring back accidentally deleted objects** (an OU full of users someone nuked). You boot into **Directory Services Restore Mode (DSRM)** using that DSRM password you set during promotion (§5.4).

```text
# Recover deleted objects (authoritative restore), simplified:
 1. Boot the DC into DSRM (or use the AD Recycle Bin instead — see below, it's far easier).
 2. wbadmin start systemstaterecovery -version:<id>     (restore system state)
 3. ntdsutil -> "authoritative restore" -> "restore subtree OU=Sales,DC=contoso,DC=local"
 4. Reboot; the restored objects out-version and re-replicate to all DCs.
```

> **⚡ Modern best practice — AD Recycle Bin:** Enable the **AD Recycle Bin** (`Enable-ADOptionalFeature 'Recycle Bin Feature' ...`) and recovering a deleted user becomes a one-liner (`Restore-ADObject`) with all attributes intact — no DSRM, no authoritative restore needed for the common case. Enable it *before* you need it (it's irreversible to disable but harmless to have on).

```powershell
# Enable the AD Recycle Bin (forest-wide; requires 2008 R2+ functional level):
Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet `
  -Target (Get-ADForest).Name
# Later, undelete a user in one line:
Get-ADObject -Filter 'Name -eq "Jane Doe"' -IncludeDeletedObjects | Restore-ADObject
```

### 15.4 Shadow copies as fast recovery **[I]**

For accidental file deletes/overwrites, **VSS shadow copies** (§8.5) restore in seconds via *Previous Versions* — far faster than a backup restore. They're a complement, not a replacement (same-volume = not disaster-proof).

### 15.5 A pragmatic DR plan outline **[I/A]**

| Element | What to define |
|---|---|
| **Inventory & priorities** | Which servers/services are tier-0 (DCs, DNS), tier-1 (line-of-business), tier-2? Restore order. |
| **RPO/RTO per tier** | Backup frequency + restore method to meet each. |
| **Backup design** | 3-2-1(+immutable); what's backed up, retention, where the offsite/air-gapped copy lives. |
| **Runbooks** | Step-by-step restore for each scenario (single server, DC, whole-site, ransomware). |
| **Test schedule** | Periodic **test restores** + at least one tabletop/failover exercise per year. |
| **Contacts & dependencies** | Who to call, vendor support, the order things must come up (DNS/DC first). |

> **Security:** Treat backups as a prime ransomware target. Keep at least one copy **immutable or offline/air-gapped**, restrict backup-system credentials (separate from domain admin), and **alert on backup failures** — silent failed backups are how "we have backups" becomes "we had backups."

---

## 16. Automation & At-Scale Management — Scheduled Tasks, DSC, WAC, Azure Arc

### 16.1 Scheduled tasks **[A]**

The **Task Scheduler** runs scripts/programs on a schedule or trigger (time, boot, logon, an event ID firing). It's the workhorse for routine automation — log cleanup, report generation, custom monitoring. Define tasks in PowerShell so they're repeatable across servers:

```powershell
$action  = New-ScheduledTaskAction -Execute "pwsh.exe" `
             -Argument "-NoProfile -File C:\Scripts\NightlyReport.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 1:30AM
# Run as a gMSA or a dedicated low-priv service account, with highest privileges only if needed:
$principal = New-ScheduledTaskPrincipal -UserId "CONTOSO\svc_tasks$" -LogonType Password -RunLevel Highest
Register-ScheduledTask -TaskName "Nightly Report" -Action $action -Trigger $trigger -Principal $principal
Get-ScheduledTask -TaskName "Nightly Report" | Get-ScheduledTaskInfo   # last run / next run / result
```

> **Best practice:** Don't store passwords in tasks — run under a **gMSA** (`$` accounts) so AD rotates the secret. Log task output to a file and **monitor for failures** (§14.5).

### 16.2 Desired State Configuration (DSC) **[A]**

**DSC is configuration-as-code for Windows.** Instead of clicking or running setup scripts that drift over time, you *declare* the desired end state ("IIS installed, this site present, this service running, this registry value set") in a **configuration**, compile it to a **MOF**, and the **Local Configuration Manager (LCM)** on each server **continuously enforces** it — auto-correcting drift. This is how you keep 200 web servers *provably identical* and self-healing. (Conceptually it's the Windows cousin of Ansible/Puppet; **⚡ Version note:** classic PSDSC v1/v2 still ships, and **DSC v3** is the modern cross-platform reboot — and Azure offers **Machine Configuration** via Arc as the cloud-managed successor.)

```powershell
Configuration WebServerBaseline {
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    Node "WEB01" {
        WindowsFeature IIS    { Name = "Web-Server"; Ensure = "Present" }
        WindowsFeature ASP    { Name = "Web-Asp-Net45"; Ensure = "Present" }
        Service W3SVC         { Name = "W3SVC"; State = "Running"; StartupType = "Automatic" }
        File SiteRoot         { DestinationPath = "C:\inetpub\contoso"; Type = "Directory"; Ensure = "Present" }
    }
}
WebServerBaseline -OutputPath C:\DSC                 # compile to MOF
Start-DscConfiguration -Path C:\DSC -Wait -Verbose   # apply
Test-DscConfiguration                                # are we still in the desired state? (drift check)
```

### 16.3 Windows Admin Center at scale & PowerShell as the API **[A]**

For mixed estates, **WAC** centralizes day-to-day ops across many servers (and Server Core) from a browser. But the deepest automation lever remains **PowerShell remoting + a source-controlled script library**: treat your admin scripts like code (in Git — see [Git](GIT_GUIDE.md) practices generally), parameterize them, target server lists pulled from AD (§3.6), and emit objects you can report on. Combine with **scheduled tasks** or a CI runner to run them unattended.

### 16.4 Hybrid & Azure Arc — the modern direction **[A]**

**Azure Arc** projects on-prem (and other-cloud) Windows Servers into Azure as managed resources, letting you apply **Azure Policy**, **Update Manager** (patching), **Machine Configuration** (DSC-style guest config), **Defender for Cloud**, and **Monitor** to servers sitting in your own datacenter — one control plane for on-prem + cloud. **⚡ Version note:** Windows Server 2025 leans into this with **"pay-as-you-go" licensing via Arc**, **hotpatching** (install many security updates **without rebooting**, delivered through Arc), and tighter Azure integration. Even fully on-prem shops increasingly adopt Arc for unified patching, policy, and security at scale. For container workloads, see [Docker](DOCKER_GUIDE.md); for Linux halves of the fleet, [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md).

---

## 17. Production Operations & Company-Level Practices

> Tools get you a working server; **practices** get you a *reliable, audited, secure* environment that survives staff turnover, audits, and incidents. This is what "company level" actually means.

### 17.1 Change control **[A]**

In production you don't just "make a change." Every non-trivial change goes through **change control**: a written **request** (what/why/risk/rollback), **approval** (a change advisory board for risky changes), a **scheduled window**, a **test/pilot** first where possible, **execution with a backout plan**, and **post-change verification**. This prevents the Friday-afternoon GPO edit that locks out 5,000 users with no record of who did it or how to undo it. Even a small shop benefits from a lightweight change log.

### 17.2 Patch cadence **[A]**

Operationalize §13.7: a defined monthly **Patch Tuesday** cadence, **ring-based rollout** (pilot → broad), **always after a verified backup**, inside a **maintenance window**, with **post-patch health checks** and a rollback plan. Track which servers are at which patch level (report via `Get-HotFix`/Update Manager). Out-of-band/critical patches get an expedited but still-controlled path.

### 17.3 The tiered admin model & privileged access **[A]**

The most important enterprise-security practice. Microsoft's **tiered administration model** splits admin identities so a compromise can't escalate:

| Tier | Controls | Rule |
|---|---|---|
| **Tier 0** | Domain Controllers, AD, PKI, identity | Tier-0 admin accounts log on **only** to Tier-0 systems. |
| **Tier 1** | Servers & applications | Tier-1 admins manage servers, **never** log into DCs or workstations as admin. |
| **Tier 2** | Workstations & devices | Helpdesk/desktop admins, **never** touch servers/DCs with privileged creds. |

The golden rule: **a privileged credential must never be exposed on a lower-trust machine**, because that machine could be compromised and harvest it. Enforce with: **separate admin accounts** (never your daily account), **Privileged Access Workstations (PAWs)** for Tier 0/1 admin work, **Protected Users** + Credential Guard + LAPS (§13), **Just-In-Time** elevation (PIM/PAM where available), and logon-restriction GPOs. This single model breaks the "phish a helpdesk laptop → reach a DC" kill chain.

### 17.4 Documentation & runbooks **[A]**

Operations that live only in one admin's head are a liability. Maintain: an **asset/inventory** record (servers, roles, IPs, owners, dependencies), **network and AD diagrams**, **runbooks** (step-by-step procedures: onboard a user, build a server, restore a DC, fail over DHCP), **a config baseline** (ideally as code — DSC/scripts in Git), and a **change log**. When the on-call person at 3 a.m. isn't you, the runbook is the difference between a 10-minute and a 6-hour outage.

### 17.5 Standardization, monitoring & compliance **[A]**

- **Standardize builds.** Every server from the same image/script (DSC/answer file), same baseline, same naming/IP convention. Snowflakes are unmanageable and insecure.
- **Monitor & alert** on the §14.5 signals; route alerts somewhere a human sees them; review them.
- **Compliance.** Map your controls to whatever framework applies (CIS, ISO 27001, SOC 2, PCI-DSS, HIPAA). Security baselines (§13.5), auditing (§13.8), and documented change/patch processes are exactly what auditors ask for. Keep evidence (logs, approvals, baseline reports).
- **Capacity & lifecycle.** Track resource trends (§14.3) for capacity planning, and track OS lifecycle — **don't run out-of-support Windows Server** (no security updates = automatic audit finding and breach risk).

---

## 18. Gotchas & Best Practices

### 18.1 The classic foot-guns **[I/A]**

- **Pointing a domain member's DNS at a public resolver** (8.8.8.8) instead of your DCs → domain join fails, Group Policy and logon silently break. **Members → DCs for DNS, always** (§2.3).
- **Servers on DHCP.** Use static IPs (or reservations). A server that moves address breaks everything depending on it.
- **Clock drift > 5 min** → Kerberos fails, "access denied" everywhere. Keep time synced from the PDC-Emulator (§2.3, §5.6).
- **Confusing Share vs NTFS permissions.** Effective access = **most restrictive of the two**. Set Share to `Authenticated Users = Full Control`, gate with NTFS (§4.3). Forgetting `(OI)(CI)` inheritance flags is the #1 ACL bug.
- **OUs vs groups.** OUs target policy/delegation; **groups grant permissions**. Don't try to "give the Sales OU access to a folder" (§5.2).
- **Seizing an FSMO role then reviving the old holder** → split-brain. Seize only when the old DC is gone forever (§5.6).
- **Reverting a DC from a checkpoint/snapshot** → USN rollback / replication corruption. Use VM-GenerationID-aware hosts and proper AD backups instead (§10.4, §15.3).
- **Leaving Hyper-V checkpoints around** → ballooning differencing disks, slow VMs. Delete promptly (§10.4).
- **Disabling the firewall or UAC "to make it work."** You've just removed two of your best defenses. Open the specific port / elevate deliberately instead (§4.4, §12.1).
- **Re-enabling SMB1** for an old device. It's wormable (WannaCry/NotPetya). Replace the device (§8.1).
- **One giant "do everything" GPO.** Impossible to troubleshoot or delegate. Small, purpose-named, tested-in-pilot GPOs (§6.6).
- **`Block Inheritance` surprises** — `Enforced` GPOs punch through it. Map your RSoP before assuming what applies (§6.5).
- **Same local admin password everywhere** → one box owned = all owned. Deploy **LAPS** (§13.4).
- **Daily-driver account in Domain Admins** and browsing the web from a DC → instant game-over on a phish. **Tiered admin + PAW** (§17.3).
- **Backups never test-restored**, or the only copy is online/encryptable by ransomware. Test restores; keep an immutable/offline copy (§15).
- **Running out-of-support Windows Server.** No patches = breach + audit failure. Track lifecycle (§17.5).

### 18.2 Best-practice recap **[I/A]**

- **Default to Server Core** for infrastructure roles; manage remotely (WAC/RSAT/PowerShell); keep a Desktop box only where a vendor demands it.
- **Two DCs minimum**, AD-integrated DNS, accurate time, **AD Recycle Bin enabled**, **at least daily system-state backup** of DCs.
- **Least privilege everywhere:** separate admin accounts, groups-not-individuals for permissions, **JEA** and **delegation** for narrow tasks, **gMSA** for services.
- **Everything as code/script** where you can (DSC, repeatable PowerShell, GPO backups, source control) — reproducibility is reliability *and* security.
- **Apply a security baseline** (SCT/CIS) via GPO; layer LAPS, Credential Guard, Protected Users, ASR, BitLocker; **patch monthly in rings after backups**.
- **Monitor & forward logs** centrally; alert on failed logons, privileged-group changes, backup failures, low disk.
- **Document and change-control.** Runbooks, diagrams, inventory, a change log — so the environment survives you.
- **Test your DR.** A backup you haven't restored is a rumor.

---

## 19. Study Path & Build-to-Learn Projects

### 19.1 A suggested learning order

1. **[B] OS fundamentals & install.** Install Windows Server 2025 **twice** — once **Desktop Experience**, once **Server Core** — in VMs (Hyper-V, VirtualBox, or VMware). Do the whole §2 initial-config checklist by hand on Desktop, then **entirely in PowerShell** on Core. Get comfortable that Core has no GUI and you survive fine.
2. **[B] PowerShell as your hands.** Work through §3 against both VMs: the object pipeline, `Get-Help`/`Get-Member`/`Get-Command`, then **remoting** — `Enter-PSSession` and `Invoke-Command` from Desktop *to* Core. This is the skill everything else rides on; cross-train with the [PowerShell](POWERSHELL_GUIDE.md) guide.
3. **[B/I] Local security model.** Create local users/groups, build a share, and *prove* to yourself how **Share vs NTFS** permissions combine (§4.3) by logging in as different users and watching access change. Break it, fix it, understand "most restrictive wins."
4. **[I/A] Stand up Active Directory.** Promote your Desktop VM to **DC01** (new forest), add **DC02** for redundancy, point DNS correctly, and **join** a member server and a Windows 11 client. Install **RSAT** on the client and explore ADUC, DNS, DHCP, GPMC. This is the centerpiece — take your time.
5. **[I/A] Run the directory.** Bulk-create users from CSV (§6.2), build an OU structure, do **AGDLP** grouping, **delegate** password resets to a "helpdesk" account, and write your first **GPOs** (lock screen, firewall, drive map) — then *predict* RSoP with `gpresult` before checking (§6.5).
6. **[I] Core infra roles.** Configure **DNS** records, a **DHCP** scope with reservations and **failover**, and a governed **file server** (SMB encryption, ABE, FSRM quota, DFS-N namespace, VSS). Map a drive via GPO and watch it appear on the client.
7. **[I/A] Web & virtualization.** Host a site in **IIS** (app pool, binding, TLS cert from an internal CA), then enable **Hyper-V** *nested* and spin up a VM inside your VM — make a checkpoint, break something, roll back.
8. **[I/A] Storage.** Add disks, build a **Storage Spaces** mirror pool, format **ReFS**, and present an **iSCSI** LUN from one server to another. Understand GPT, resiliency trade-offs, ReFS vs NTFS.
9. **[A] Harden it.** Apply a **Microsoft Security Baseline** via GPO, deploy **LAPS**, add admins to **Protected Users**, enable an **ASR** rule (audit → block), turn on **BitLocker**, set up **auditing** + **event forwarding** to a collector. Re-run Policy Analyzer to measure drift.
10. **[I/A] Back it up & break it.** Configure **Windows Server Backup** (system state + BMR), enable the **AD Recycle Bin**, then **simulate disasters**: delete an OU and recover it (Recycle Bin *and* the hard way via authoritative restore), kill a DC and rebuild it non-authoritatively, restore a file via Previous Versions. Write the **runbook** as you go.
11. **[A] Automate & operate like a company.** Wrap routine work in **scheduled tasks** (running as a **gMSA**), enforce a server's config with **DSC** (drift-correct it on purpose), centralize management in **Windows Admin Center**, and read up on **Azure Arc**. Then put on your "company" hat: write a change-control note, a patch-ring plan, a tiered-admin diagram, and a one-page DR plan.

### 19.2 Build-to-learn projects (build the lab: a DC + a member server)

- **Project 1 — The foundation lab.** Two VMs minimum: **DC01** (DC + DNS + DHCP, Server Core if you're brave) and **MS01** (member server, Desktop or Core) plus a **Windows 11 client**. Static IPs, correct DNS, domain `lab.contoso.local` (or a subdomain you own). Everything else builds on this. *Deliverable: a working domain you can log into from the client with a domain account.*
- **Project 2 — Identity & policy.** Bulk-import 50 users + groups from CSV with AGDLP grouping; build a real OU tree; create 4–5 purpose-named GPOs (lock screen, firewall baseline, drive maps, blocked USB, admin logon restriction); delegate helpdesk password resets. *Deliverable: a `gpresult` HTML report proving the right policies hit the right machines, and a screenshot of a delegated reset working as a non-admin.*
- **Project 3 — File & print services done right.** A DFS namespace `\\lab.contoso.local\files` fronting an encrypted, ABE-enabled SMB share on MS01 with an FSRM quota and a media file-screen, replicated (DFS-R) to a second target, with VSS shadow copies. *Deliverable: move the data to a "new" server and prove users' mapped drive path never changed.*
- **Project 4 — Web + PKI.** Stand up AD Certificate Services (or a self-signed cert), host an IIS site over **HTTPS** with a proper app pool/gMSA identity, then put **Nginx** (a Linux VM or container) in front as a reverse proxy/TLS terminator — cross-reference the [Nginx](NGINX_GUIDE.md) and [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md) guides. *Deliverable: a working HTTPS app reachable through the proxy.*
- **Project 5 — Resilience.** Configure **DHCP failover** between DC01 and a second server, add a **second DC**, build a **Storage Spaces mirror**, and present an **iSCSI** LUN. Kill one of each (DC, DHCP, disk) and show the service survives. *Deliverable: an outage log showing what failed and that clients kept working.*
- **Project 6 — Harden & audit.** Apply the Windows Server 2025 **security baseline**, deploy **LAPS**, enable **Credential Guard** + **Protected Users**, switch on **ASR** + auditing, forward events to a collector, and **BitLocker** a data drive. *Deliverable: a Policy Analyzer before/after diff and a sample of forwarded 4625/4688 events on the collector.*
- **Project 7 — DR drill.** Full backup of DC01 (system state + BMR). Then: (a) delete an OU of users and recover via **AD Recycle Bin**; (b) recover the *same* deletion the hard way via **authoritative restore**; (c) destroy DC01 and rebuild it **non-authoritatively** from DC02; (d) restore a file via **Previous Versions**. *Deliverable: a tested **runbook** with timings — your real RTO/RPO numbers.*
- **Project 8 — Operate at scale.** Write a PowerShell "fleet report" that pulls every server from AD and reports uptime, free disk, pending reboots, last patch, and backup status as one table; schedule it (gMSA); enforce a baseline config on MS01 with **DSC** and watch it auto-correct drift you introduce; manage it all from **Windows Admin Center**. *Deliverable: the report, the DSC config in a Git repo, and a one-page change-control + tiered-admin write-up.*

---

*End of the Windows Server Administration reference. Cross-references: **POWERSHELL_GUIDE** (the language that runs everything here), **WINDOWS_CMD_BATCH_GUIDE** (the legacy shell), **NETWORKING_GUIDE** (IP/DNS/TLS/Kerberos fundamentals), **NGINX_GUIDE** (reverse proxy in front of IIS/app servers), **DOCKER_GUIDE** (Windows containers), **DATABASE_SERVER_ADMIN_GUIDE** (SQL Server on Windows), and **LINUX_SERVER_ADMIN_GUIDE** (the other half of nearly every real fleet). Build the DC-plus-member-server lab, break it on purpose, restore it, and harden it — that loop, not reading, is how Windows Server mastery is actually built.*
