# HERMES — Helpdesk Ticketing and Automation

HERMES is a simulated IT helpdesk environment built on an Active Directory domain with a live ticketing platform. Every ticket is worked end-to-end — submitted via osTicket, triaged, resolved with documented steps and PowerShell automation where appropriate, and closed with user confirmation.

---

## Lab Environment

| Hostname | IP Address | OS | Role |
|---|---|---|---|
| DC-01 | 192.168.0.10 | Windows Server 2022 | Domain Controller, DNS, AD DS |
| WS-01 | 192.168.0.70 | Windows 10 Pro | Domain-joined workstation |
| INFRA-01 | 192.168.0.50 | Ubuntu 24.04 | osTicket helpdesk platform |
| WAZUH-01 | 192.168.0.60 | Ubuntu 22.04 | Wazuh SIEM / XDR |
| KALI-01 | 192.168.0.80 | Kali 2026.1 | Attack simulation |

**Domain:** `lab.local` · **Scripts:** `C:\Scripts\` on DC-01 · **Username format:** `firstname.lastname`

---

## Domain Security Policy

| Policy | Value |
|---|---|
| Minimum password length | 14 characters |
| Password complexity | Enabled |
| Account lockout threshold | 3 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset lockout counter after | 30 minutes |

Policy applied to: Default Domain Policy — `DC01.LAB.LOCAL`

![Get-ADDefaultDomainPasswordPolicy — complexity enabled, lockout threshold 3, min length 14] <img width="697" height="291" alt="image" src="https://github.com/user-attachments/assets/54b89beb-1d9d-41c6-a59d-94e3e29b8a3c" />


---

## OU Structure

```
lab.local
└── Departments
    ├── Finance
    │   └── Users
    ├── HR
    │   └── Users
    ├── IT
    │   └── Users
    └── Sales
        └── Users
```

Provisioned using `New-OUStructure.ps1`. Finance OU existed from earlier lab work; IT, HR, and Sales were created fresh.

![ADUC — all four department OUs confirmed] <img width="662" height="299" alt="image" src="https://github.com/user-attachments/assets/18ce659a-44e7-418d-aaa0-3e3d5888841b" />


---

## Security Groups and Access Model

| Group | Members | Access Granted |
|---|---|---|
| `Finance-Users` | All Finance users | Read/Write — `\\DC-01\Shares\Finance` |
| `HR-Users` | All HR users | Read/Write — `\\DC-01\Shares\HR` |
| `IT-Users` | All IT users | Read/Write — `\\DC-01\Shares\IT` |
| `Sales-Users` | All Sales users | Read/Write — `\\DC-01\Shares\Sales` |
| `Dept-Managers` | All four department managers | Scoped AD delegation over their department OU |
| `Helpdesk-Admins` | Paul Atreides, Michael Cabrera, Nazeer England, Olivia Fowler | Delegated AD control across all OUs + Read on all shares |

No permissions assigned directly to users — all access flows through group membership.

---

## Tickets

### [T001 — Password Reset](T001-password-reset.md)
**Ticket #530759 · Priority: Normal · Closed**

- User unable to log into WS-01; ticket submitted on her behalf from DC-01 (no workstation access)
- Password reset via ADUC — forced change at next logon enforced, account unlocked in same action
- Temp password communicated by phone — never stored in ticket or sent via email
- User confirmed successful login; ticket closed after validation

---

### [T002 — Account Lockout](T002-account-lockout.md)
**Ticket #170269 · Priority: Normal · Closed**

- User hit the 3-attempt lockout threshold and submitted the ticket herself from WS-01
- Lockout confirmed via `Get-ADUser` before taking any action
- Account unlocked using `Unlock-ADAccount.ps1` — script displays account details, bad logon count, and next steps on completion
- User replied in the ticket thread confirming access restored

![Unlock-ADAccount.ps1 — jasmine.rodgers, LockedOut: True, Bad Logons: 3, unlocked successfully] <img width="894" height="351" alt="image" src="https://github.com/user-attachments/assets/4d02b527-59a5-4b1f-a699-d29b53b9a8ad" />


---

### [T003 — New User Onboarding](T003-New%20User%20Onboarding.md)
**4 tickets · 15 users onboarded across Finance, HR, IT, Sales**

- Four department managers submitted separate onboarding requests via osTicket
- Environment stood up in sequence before any accounts were created:
  - `New-OUStructure.ps1` — provisioned IT, HR, Sales OUs (Finance already existed)
  - `New-DeptLeaders.ps1` — created all four manager accounts
  - `New-SecurityGroups.ps1` — created 6 groups, 0 skipped, 0 failed
- Department users created with four dedicated scripts — one per department
  - Each script: correct OU placement, group assignment, temp password, forced change at first logon
  - IT users additionally assigned to `Helpdesk-Admins` (delegated AD control, no Domain Admin rights)
- Login validation on WS-01 completed for Finance and HR users; group membership in script output verified IT and Sales
- First run of `New-DeptLeaders.ps1` failed — `TempPass123!` didn't meet the 14-character domain minimum; documented and resolved

![New-DeptLeaders.ps1 — password complexity error on all four accounts, first run] <img width="959" height="479" alt="image" src="https://github.com/user-attachments/assets/ab6593f4-cece-4da5-b585-1128178183ec" />

![New-SecurityGroups.ps1 — 6 groups created, 0 skipped, 0 failed] <img width="852" height="449" alt="image" src="https://github.com/user-attachments/assets/4d596a35-8f6a-4cee-ad4d-ba6e4ac6b07a" />

---

### [T004 — Unable to Access IT Network Share](T004-Network-share-access.md)
**Ticket #779507 · Priority: High · Closed · Open-to-close: 4 minutes**

- michael.cabrera unable to access `\\DC-01\IT` from WS-01 — "network path not found"
- paul.atreides (ticket submitter) found locked out and disabled before troubleshooting could begin — re-enabled via ADUC, unlocked via `Unlock-ADAccount.ps1`
- Share availability verified on DC-01 first (`Get-SmbShare`) before touching the workstation
- Workstation-level tests run as michael.cabrera:
  - `\\DC-01\IT` → failed · `\\192.168.0.10\IT` → failed · `ping DC-01` → failed · `ping 192.168.0.10` → 4/4 success
  - SMB port 445 open · shares visible via `net view \\192.168.0.10`
- Root cause: WS-01 NIC not pointing to DC-01 as DNS server — hostname resolution failing
- Resolution: IT share manually mapped as Z: via `net use` — temporary fix, DNS misconfiguration documented as follow-up

![WS-01 — UNC failures, ping DC-01 fails, ping IP succeeds] <img width="677" height="483" alt="image" src="https://github.com/user-attachments/assets/893399f4-2757-446f-8dbc-ef555f0920f9" />


![This PC — IT (\\192.168.0.10) mapped as Z: and accessible] <img width="782" height="528" alt="image" src="https://github.com/user-attachments/assets/ebb7d3ea-acb6-4ec7-a76f-60caf6f345e7" />


---

### [T005 — PowerShell Execution Policy Enforcement via GPO](T005-Gpo%20powershell%20execution%20policy.md)
**Security Control Implementation · Scope: All domain workstations**

- GPO (`Workstation-PS-ExecutionPolicy`) created and linked to Departments OU
- Enforces `AllSigned` at Computer Configuration level — applies regardless of logged-in user, cannot be overridden locally
- Verified via `gpresult /r` — policy confirmed applied from `DC01.lab.local`
- Live enforcement test: unsigned script on WS-01 returned `PSSecurityException: UnauthorizedAccess`
- Maps to MITRE ATT&CK T1059.001 — PowerShell abuse as execution vector

![gpresult /r — Workstation-PS-ExecutionPolicy confirmed applied on WS-01] <img width="599" height="211" alt="image" src="https://github.com/user-attachments/assets/90df61e6-2342-42bb-aef2-e4b9450ab010" />


![WS-01 — unsigned script blocked: PSSecurityException, UnauthorizedAccess] <img width="791" height="164" alt="image" src="https://github.com/user-attachments/assets/696bce15-03c9-46ff-becf-e2890865c961" />


---

## PowerShell Scripts

Full scripts with line-by-line explanations are embedded in each ticket document. For a quick reference of all scripts, parameters, and usage — see [SCRIPTS.md](SCRIPTS.md).

| Script | Used In | Purpose |
|---|---|---|
| `Unlock-ADAccount.ps1` | T002, T004 | Checks lockout status and unlocks; displays account details and bad logon count |
| `New-OUStructure.ps1` | T003 | Provisions the Departments OU and all four department sub-OUs |
| `New-DeptLeaders.ps1` | T003 | Creates the four department manager accounts |
| `New-SecurityGroups.ps1` | T003 | Creates the six security groups and seeds group membership |
| `New-FinanceUser.ps1` | T003 | Creates a Finance user, assigns to `Finance-Users`, sets temp password |
| `New-HRUser.ps1` | T003 | Creates an HR user, assigns to `HR-Users`, sets temp password |
| `New-ITUser.ps1` | T003 | Creates an IT user, assigns to `IT-Users` and `Helpdesk-Admins`, sets temp password |
| `New-SalesUser.ps1` | T003 | Creates a Sales user, assigns to `Sales-Users`, sets temp password |
| `New-DeptShares.ps1` | T004 | Creates the four department SMB shares with NTFS and share-level permissions |

---

## Foundation

HERMES runs on top of an osTicket deployment with Active Directory authentication integration. Platform setup — installation, AD auth via LDAP, agents, departments, SLAs, and help topics — is documented in the [osTicket Help Desk Deployment with Active Directory Authentication](https://github.com/AdemolaTech23/osTicket-Help-Desk-Deployment-with-Active-Directory-Authentication) repo. Every ticket in HERMES was worked through that live platform.
