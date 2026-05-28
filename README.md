# AD-Lab
Windows Server 2025 Active Diectory lab in VirtualBox: domain controller, DHCP, DNS, group policy, and break/fix scenarios.

## Topology
- **DC01** — Windows Server 2025, Domain Controller, DNS, DHCP
- **CLIENT01** — Windows 10/11, domain-joined
- **CLIENT02** — Windows 10/11, domain-joined
- Network: VirtualBox Internal Network (`LAB-NET`), 192.168.10.0/24
- Domain: `lab.local`

## Build log
- Phase 1: Server deployment and base config
- Phase 2: AD DS promotion
- Phase 3: DHCP scope
- Phase 4: Client deployment and domain join
- Phase 5: OU structure, users, groups (PowerShell bulk creation)
- Phase 6: Group Policy
- Phase 7: Break/fix scenarios

## Tools used
Windows Server 2025, VirtualBox 7.x, PowerShell, ADUC, GPMC, DHCP console