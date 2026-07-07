# AD-Lab — Windows Server 2025 Active Directory Lab

A fully functional Active Directory environment built from scratch in
VirtualBox: domain controller deployment, DNS, DHCP, user and group
administration, Group Policy, and hands-on help desk troubleshooting. Every
phase is documented with commands, screenshots, and the reasoning behind each
decision.

**Full build documentation:** [deployment.md](01-Deployment/deployment.md)

## Topology
- **DC01** — Windows Server 2025 — Domain Controller, DNS, DHCP (192.168.10.10)
- **CLIENT01** — Windows 11 Pro — domain-joined
- **Network:** VirtualBox Internal Network (`LAB-NET`), 192.168.10.0/24
- **Domain:** `lab.local`

## What This Demonstrates
- Deploying a Windows Server 2025 domain controller (AD DS, DNS, DHCP)
- Deploying and domain-joining a Windows 11 Pro client
- User and group administration, including **bulk user provisioning via PowerShell**
- OU design and **Group Policy** to control users and machines
- Diagnosing and resolving real help desk tickets with a repeatable
  **diagnose → fix → verify** methodology

## Build Phases
1. **Server deployment & base config** — static IP, hostname, isolated network
2. **AD DS promotion** — new forest `lab.local`, DNS integrated
3. **DHCP** — authorized scope, automatic client addressing
4. **Client deployment & domain join** — Windows 11 Pro joined to the domain
5. **OUs, users & groups** — structure, bulk CSV user creation, access model
6. **Group Policy** — domain password policy + OU-scoped user restrictions
7. **Break/fix scenarios** — account lockout, disabled account, lost access

## Troubleshooting Highlights
Real problems diagnosed and resolved during the build:
- **DNS/SRV registration failure** blocking domain join — traced through
  layered diagnosis (IP reachable → name unresolved → missing A record →
  missing SRV records) and fixed via manual registration + Netlogon restart
- **Group Policy password-policy quirk** — password policy enforces only at the
  domain level, not per-OU; identified and corrected
- **Access-token behavior** — why restored group membership requires re-login

## Tools
Windows Server 2025 · Active Directory · DNS · DHCP · Group Policy ·
PowerShell · ADUC · GPMC · VirtualBox · Windows 11 Pro