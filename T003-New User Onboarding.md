# T003 — New User Onboarding
 
## Ticket Details
 
| Field | Details |
|---|---|
| Ticket ID | TBD |
| Date | 2026-04-16 |
| Submitted via | osTicket (web portal) |
| Priority | Normal |
| Status | In Progress |
| Assigned To | Ademola Durodola |
| Environment | DC-01 (Windows Server 2022) · WS-01 (Windows 10 Pro) · INFRA-01 (osTicket) |
 
---
 
## User Report
 
A manager submits a ticket requesting a new employee account to be created ahead of their start date.
 
> "Hi, we have a new team member joining the Finance department. Can you please set up their domain account so they are ready to log in on their first day?"
 
*Screenshot coming soon*
 
---
 
## Prerequisites — OU Structure
 
Before new user accounts can be created in the correct department, a standardised Organisational Unit structure was built on DC-01 using `New-OUStructure.ps1`.
 
**Structure created:**
 
```
lab.local
└── Departments
    ├── Finance
    │   └── Users
    ├── IT
    │   └── Users
    ├── HR
    │   └── Users
    └── Sales
        └── Users
```
 
**Script run on DC-01:**
 
```powershell
.\New-OUStructure.ps1
```
 
Script output confirmed:
- Departments OU already existed — skipped
- Finance OU already existed — skipped
- IT, HR, and Sales OUs created successfully with Users sub-OUs
