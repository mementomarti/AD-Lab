# Deployment & Domain Controller Promotion

## Lab Environment
- **Host:** VirtualBox 7.x
- **Server OS:** Windows Server 2025 Standard (Desktop Experience)
- **VM specs:** 4 GB RAM, 2 vCPU, 60 GB disk (dynamic), EFI enabled
- **Network:** VirtualBox Internal Network (`LAB-NET`)
- **Domain:** `lab.local` (NetBIOS: `LAB`)

---

## Phase 1: Base Server Configuration

Renamed the server and assigned a static IP before promoting it to a domain controller.

| Setting | Value |
|---|---|
| Hostname | `DC01` |
| IP address | `192.168.10.10` |
| Subnet mask | `255.255.255.0` (/24) |
| Default gateway | none (isolated lab network) |
| Preferred DNS | `127.0.0.1` (loopback — server is its own DNS) |

**Why a static IP:** A domain controller must have a fixed address. Clients
locate the DC by its IP, and Active Directory + DNS break if that address
changes. DNS points to loopback because once promoted, DC01 *is* the DNS
server for the domain.

![Static IP configuration](screenshots/p1-01-ipv4-config.png)
![Rename to DC01](screenshots/p1-02-rename.png)

Verified with `hostname` and `ipconfig`, then took a snapshot (`pre-ADDS`)
as a rollback point.

---

## Phase 2: Active Directory Domain Services Promotion

### Role installation
Installed the **Active Directory Domain Services** and **DNS Server** roles
via Server Manager → Add Roles and Features.

![AD DS role selected](screenshots/p2-01-adds-role.png)
![AD DS installed](screenshots/p2-02-adds-installed.png)

### Promotion
Promoted DC01 to a domain controller in a **new forest**.

| Setting | Value |
|---|---|
| Deployment | New forest |
| Root domain | `lab.local` |
| Functional level | Windows Server 2025 |
| DNS server | Enabled |
| DSRM password | Set (recovery use) |

![New forest](screenshots/p2-04-new-forest.png)
![DC options](screenshots/p2-05-dc-options.png)
![Prerequisites passed](screenshots/p2-07-prereq.png)

The server rebooted automatically and came back showing `LAB\Administrator`
at login confirming the domain was live.

![Domain controller dashboard](screenshots/p2-post-ADDS.png)

---

## Verification

Confirmed the domain controller was healthy three independent ways:

**1. DNS resolution**

nslookup lab.local -> 192.168.10.10
(A reverse lookup timeout on '::1' appeared first, which was expected since no reverse zone is configured. The forward lookup succeeded, which is the test that matters for this lab.)

**2. AD domain query**

Get-ADDomain -> DNSRoot: lab.local | NetBIOS: LAB | Mode : Windows2025Domain

**3. Domain controller diagnostics**

dcdiag -> all functional tests passed
(Connectivity, Advertising, Services, Replications, RidManager, NetLogons)

---

## Troubleshooting Note: dcdiag SystemLog & DFS-REvent Failures

Initial 'dcdiag' reported two failed tests: **SystemLog** and **DFS-REvent**.

**Diagnosis:**

-**SystemLog** flagged "unexpected shutdown events. Root cause: forced VM power-offs during early setup (mouse/display troubleshooting), which the DC logged as unclean shutdowns. Also included a transient KDC/SAM initialization error normal on freshly promoted DC.

-**DSFREvent** flagged SYSVOL replication events. Root cause: normal first-time SYSVOL initialization on a brand-new single domain controller (no replication partner exists yet).

**Resolution:** Performed a clean reboot from inside Windows and re-ran diagnostics. All functional tests passed. The flagged events were historical log entries that age out of the 24-hour scan window. Confirmed benign.

**Lesson:** Always shut down the VM from inside the OS (Start -> Power) or via ACPI Shutdown, never VirtualBox's power-off/reset, which a domain controller records as a crash.

## Phase 3: DHCP Server

Installed and configured DHCP so client VMs receive IP addresses, DNS, and domain settings automatically on boot, matching how a real network operates.

### Role Installation
Installed the DHCP role via PowerShell:

    Install-WindowsFeature -Name DHCP -IncludeManagementTools

### Authorization in Active Directory
A DHCP server must be authorized in AD before it can lease addresses. This prevents rogue DHCP servers from disrupting the domain.

    Add-DhcpServerInDC -DnsName "DC01.lab.local" -IPAdress 192.168.10.10

The command returned an "RPC server is unavailable" warning, but the line above it cofnirmed successful authorization. 'Get-DhcpServerInDC' verified DC01 was registered. The RPC Warning was a transient post-authorization check, not a failure.

![DHCP authorized in AD](screenshots/p3-02-dhcp-authorize.png)

### Security Groups
Created the DHCP Administrators / Users groups the service expects, then restarted the servie:

    netsh dhcp add securitygroups
    Restart-Service dhcpserver

### Scope Configuration

| Setting | Value |
|---|---|
| Scope name | 'LAB-Clients' |
| Range | '192.168.10.100 - 192.169.10.200' |
| Subnet Mask | '255.255.255.0 |
|Lease duration | 8 days |
|State | Active |

    Add=DhcpServerv4Scope -Name "LAB-Clients" -StartRange 192.169.10.100 
    -EndRange 192.169.10.200 -SubnetMask 255.255.255.0 -State Active

### Scope Options
Configured the options clients receive alongside their IP:

| Option | Code | Value |
|---|---|---|
| DNS Server | 6 | '192.168.10.10' |
| DNS Domain Name | 15 | 'lab.local' |
| Router (gateway) | 3 | '192.168.10.10' |
| Lease Time | 51 | 8 days |

    Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 -DnsServer 192.168.10.10 -DnsDomain "lab.local" -Router 192.168.10.10

**Why Option 6 matters:** Pointing clients to DC01 for DNS is what allows them to locate the domain and complete domain join and login. Without it, clients get an address but can't find the domain.

![DHCP scope and options verified](screenshots/p3-03-dhcp-scope.png)

### Verification
'Get-DhcpServerv4Scope' confirmed the LAB-Clients scope was Active with the correct range; 'Get-DhcpServerv4OptionValue' confirmed all four options were set.

---

## Troubleshooting Note: Stale Server Manager Dashboard Alerts

After completing DHCP, Local Server and All Servers showed red alerts.

**Diagnosis:**
- **Events** - two Critical Kernel-Power (event ID 41) entries, indicating unclean shutdowns. Root cause: forced CM power-offs during earlier sessions. Historical, not current.
- **Services** - flagged 'InventorySvc' (Inventory and Compatibility Appraisal) as stopped. This is an Automatic (Delayed Start) telemetry service that idles when not in use. Stopped is its normal state.

**Verification:** Confirmed no genuinely-failed services by querying for auto-start services that weren't running. The query returned nothing, proving all critical services (AD DS, DNS, DHCP) were healthy.

    Get-Service | Where-Object {$_.StartType -eq 'Automatic' and $_.Status -ne 'Running'}

**Resolution:** Hid the stale Kernel-Power event alerts and started InventorySvc to clear its flag. Dashboard returned to green. Both alerts were confirmed benign before clearing.