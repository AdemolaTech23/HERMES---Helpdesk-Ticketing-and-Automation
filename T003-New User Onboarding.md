# T003 — New User Onboarding

## Ticket Details

| Field | Details |
|---|---|
| Date | 2026-04-25 |
| Submitted via | osTicket (web portal) |
| Priority | Normal |
| Status | In Progress |
| Assigned To | Ademola Durodola |
| Environment | DC-01 (Windows Server 2022) · WS-01 (Windows 10 Pro) · INFRA-01 (osTicket) |

---

## Overview

New user onboarding is one of the most common Tier 1 helpdesk tasks. A department manager submits a ticket requesting domain accounts for newly hired employees. The technician creates the accounts in Active Directory, places them in the correct department OU, assigns them to the appropriate security group, sets a temporary password, and confirms access before closing the ticket.

This document covers the full onboarding workflow across all four departments — Finance, HR, IT, and Sales — including the AD environment setup required before any user accounts could be created.

The lab follows a **least privilege model**: permissions are never assigned directly to users. Instead, users are placed in department security groups, and those groups control access to network shares and AD resources. A new user inherits exactly what their role requires — nothing more.

---

## AD Environment — Full User Roster

| Name | Username | Department | Title | Role Level |
|---|---|---|---|---|
| Katherine Stetson | katherine.stetson | Finance | Finance Manager | Manager |
| Lauren Esbrand | lauren.esbrand | Finance | Finance Analyst | Staff |
| Eve Saunders | eve.saunders | Finance | Accounts Coordinator | Staff |
| Sandra Mills | sandra.mills | HR | HR Manager | Manager |
| Adrianna Sita | adrianna.sita | HR | HR Generalist | Staff |
| Jenaiah Morris | jenaiah.morris | HR | Recruitment Coordinator | Staff |
| Paul Atreides | paul.atreides | IT | IT Manager | Manager + Helpdesk |
| Michael Cabrera | michael.cabrera | IT | IT Support Specialist | Staff + Helpdesk |
| Nazeer England | nazeer.england | IT | Systems Technician | Staff + Helpdesk |
| Olivia Fowler | olivia.fowler | IT | Help Desk Technician | Staff + Helpdesk |
| Anakin Skywalker | anakin.skywalker | Sales | Sales Manager | Manager |
| Khoran Gomez | khoran.gomez | Sales | Sales Associate | Staff |
| Camryn Tapaia | camryn.tapaia | Sales | Sales Coordinator | Staff |
| Laren Johnson | laren.johnson | Sales | Account Executive | Staff |
| Catherine Krol | catherine.krol | Sales | Sales Analyst | Staff |
| Jasmine Rodgers | jasmine.rodgers | Finance | Finance Analyst | Staff (existing) |

---

## Security Groups and Access Model

| Group | Members | Access Granted |
|---|---|---|
| `Finance-Users` | All Finance users | Read/Write — `\\DC-01\Shares\Finance` |
| `HR-Users` | All HR users | Read/Write — `\\DC-01\Shares\HR` |
| `IT-Users` | All IT users | Read/Write — `\\DC-01\Shares\IT` |
| `Sales-Users` | All Sales users | Read/Write — `\\DC-01\Shares\Sales` |
| `Dept-Managers` | All four department managers | Scoped AD permissions over their own department OU |
| `Helpdesk-Admins` | Paul Atreides, Michael Cabrera, Nazeer England, Olivia Fowler | Delegated AD control across all OUs + Read on all shares |

No user has local admin on any machine. No user is assigned permissions directly — all access flows through group membership.

---

## Prerequisites — Part 1: OU Structure

Before any user accounts could be created, a standardised Organisational Unit structure was provisioned on DC-01 using `New-OUStructure.ps1`.

```powershell
.\New-OUStructure.ps1
```

**Structure created:**

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

Finance OU already existed from earlier lab work and was skipped. IT, HR, and Sales OUs were created fresh with their Users sub-OUs.

![New-OUStructure script — header and domain root] <img width="954" height="479" alt="image" src="https://github.com/user-attachments/assets/fae0fd89-46f6-4712-8fce-abe30b1b4b48" />

![New-OUStructure script — Departments OU already exists, skipping] <img width="958" height="485" alt="image" src="https://github.com/user-attachments/assets/6dbee72b-e2dc-45ed-ba1b-37794ee88e50" />

![New-OUStructure script — IT, HR, Sales OUs created] <img width="961" height="482" alt="image" src="https://github.com/user-attachments/assets/89326208-535b-495d-8aac-b7c0b04f406e" />

![New-OUStructure script — complete] <img width="903" height="398" alt="image" src="https://github.com/user-attachments/assets/a1cf2345-574b-48eb-ae1e-216ed8aa70f2" />

![ADUC — all four department OUs verified] <img width="662" height="299" alt="image" src="https://github.com/user-attachments/assets/8a3196cf-71b5-49d8-b027-d7a2eb9211ba" />

---

## Prerequisites — Part 2: Department Leaders

With the OU structure in place, department manager accounts were created using `New-DeptLeaders.ps1`. These are the accounts that submit onboarding tickets via osTicket.

```powershell
.\New-DeptLeaders.ps1
```

**Accounts created:**

| Name | Title | Department | Username |
|---|---|---|---|
| Katherine Stetson | Finance Manager | Finance | katherine.stetson |
| Sandra Mills | HR Manager | HR | sandra.mills |
| Paul Atreides | IT Manager | IT | paul.atreides |
| Anakin Skywalker | Sales Manager | Sales | anakin.skywalker |

The first run failed — the initial temp password `TempPass123!` did not meet the domain 14 character minimum. All four accounts returned a password complexity error. The script was corrected to use `TempPassword123!` (16 characters) and rerun successfully.

![New-DeptLeaders.ps1 — first run, script header] <img width="957" height="446" alt="image" src="https://github.com/user-attachments/assets/9c5bbc10-b1a3-4985-b7e3-8eb1c2ca2004" />

![New-DeptLeaders.ps1 — first run, loop and logic] <img width="952" height="483" alt="image" src="https://github.com/user-attachments/assets/704e9dd5-a91c-4fe7-92a6-7f03372ac3f2" />

![New-DeptLeaders.ps1 — first run, OU verification and account creation logic] <img width="963" height="473" alt="image" src="https://github.com/user-attachments/assets/c71ede17-7209-4d44-8db6-c30f8ceccb27" />

![New-DeptLeaders.ps1 — first run, password error on all four accounts] <img width="959" height="479" alt="image" src="https://github.com/user-attachments/assets/045704bd-dc46-4cba-a97d-2e8b61089015" />

![New-DeptLeaders.ps1 — second run, script header with corrected password] <img width="951" height="483" alt="image" src="https://github.com/user-attachments/assets/89342329-62ee-4605-a32a-a5e70ba6a7e0" />

![New-DeptLeaders.ps1 — second run, loop and account creation logic] <img width="954" height="509" alt="image" src="https://github.com/user-attachments/assets/0825d35a-6e3c-4e5d-a1ed-a4cf7b68640b" />

![New-DeptLeaders.ps1 — second run, all accounts created successfully] <img width="932" height="481" alt="image" src="https://github.com/user-attachments/assets/f0bcd94f-4640-4ba9-9adb-da0feaf27a1c" />

After the first run failure, all four accounts were bulk-updated via inline PowerShell to ensure password compliance and force a change at first logon:

```powershell
$newPassword = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force
$users = @("katherine.stetson", "sandra.mills", "paul.atreides", "anakin.skywalker")

foreach ($user in $users) {
    Set-ADAccountPassword -Identity $user -NewPassword $newPassword -Reset
    Set-ADUser -Identity $user -ChangePasswordAtLogon $true
    Write-Host "Password updated: $user" -ForegroundColor Green
}
```

![Bulk password reset — all four manager accounts updated] <img width="798" height="375" alt="image" src="https://github.com/user-attachments/assets/4be38f01-d488-4fb9-b7a5-153a90b48479" />


**ADUC verification — each manager in their correct department OU:**

![ADUC — Finance OU showing Katherine Stetson] <img width="749" height="182" alt="image" src="https://github.com/user-attachments/assets/37eb9f11-cd0b-428f-8c34-15df49c2396c" />

![ADUC — HR OU showing Sandra Mills] <img width="752" height="219" alt="image" src="https://github.com/user-attachments/assets/6dd33c0d-af1d-418c-b812-2b94d490991d" />

![ADUC — IT OU showing Paul Atreides] <img width="749" height="279" alt="image" src="https://github.com/user-attachments/assets/82793bf4-7409-4b1d-ab92-07e76ab414fa" />

![ADUC — Sales OU showing Anakin Skywalker] <img width="691" height="334" alt="image" src="https://github.com/user-attachments/assets/7ef3543e-9488-455e-bbcb-2eca2ee737ef" />


Domain password policy confirmed via PowerShell:

```powershell
Get-ADDefaultDomainPasswordPolicy
```

![Domain password policy — 14 character minimum, complexity enabled, lockout threshold 3] <img width="697" height="291" alt="image" src="https://github.com/user-attachments/assets/f83a931b-41f8-4872-b160-d6072cf87e87" />


---

## Prerequisites — Part 3: Security Groups

With the OU structure and manager accounts in place, all department security groups were created using `New-SecurityGroups.ps1`. This script creates the six groups, places each in its correct OU, and immediately assigns existing users to their appropriate groups.

```powershell
.\New-SecurityGroups.ps1
```

**Groups created:**

| Group | OU Placement | Purpose |
|---|---|---|
| `Finance-Users` | Finance\Users | Controls access to Finance share |
| `HR-Users` | HR\Users | Controls access to HR share |
| `IT-Users` | IT\Users | Controls access to IT share |
| `Sales-Users` | Sales\Users | Controls access to Sales share |
| `Dept-Managers` | Departments | Scoped AD delegation for managers |
| `Helpdesk-Admins` | IT\Users | Full department OU delegation for IT staff |

All six groups created successfully. Zero skipped, zero failed. Existing users `katherine.stetson`, `jasmine.rodgers`, and `paul.atreides` were automatically added to their respective groups by the script.

![New-SecurityGroups.ps1 — 6 groups created, 0 failed]  <img width="852" height="449" alt="image" src="https://github.com/user-attachments/assets/aeb6f689-ed4a-4e3c-868f-5d9247871f24" />


---

## Prerequisites — Part 4: osTicket User Accounts

Each department manager was registered as an end user in osTicket so they could submit tickets via the client portal. New employees have no existing system access and cannot self-serve — the manager submits on their behalf.

| Name | Email | osTicket Role |
|---|---|---|
| Katherine Stetson | katherine.stetson@lab.local | End User |
| Sandra Mills | sandra.mills@lab.local | End User |
| Paul Atreides | paul.atreides@lab.local | End User |
| Anakin Skywalker | anakin.skywalker@lab.local | End User |

![osTicket User Directory — all four managers registered] <img width="987" height="300" alt="image" src="https://github.com/user-attachments/assets/ddca31bf-3330-4946-872b-4b8ab11029e6" />


---

## Ticket Simulations

Each department manager submitted a ticket via the osTicket client portal requesting accounts for their new hires. All tickets were assigned to and resolved by Ademola Durodola.

---

### T003-A — Finance

**Submitted by:** Katherine Stetson (`katherine.stetson@lab.local`)
**New hires:** Lauren Esbrand, Eve Saunders
**Script:** `New-FinanceUser.ps1`

Katherine submitted Ticket #408626 via the osTicket client portal requesting domain accounts for two incoming Finance team members.

![Ticket #408626 — Katherine's submission and ticket details] <img width="957" height="769" alt="image" src="https://github.com/user-attachments/assets/ab00d8eb-f294-4656-a6a8-8029102687cd" />

**Script run — Lauren Esbrand:**

```powershell
.\New-FinanceUser.ps1 -FirstName "Lauren" -LastName "Esbrand" -JobTitle "Finance Analyst"
```

![New-FinanceUser.ps1 — lauren.esbrand created successfully] <img width="756" height="247" alt="image" src="https://github.com/user-attachments/assets/ad571a6a-bd5f-4141-9839-c978a58ff047" />


**Script run — Eve Saunders:**

```powershell
.\New-FinanceUser.ps1 -FirstName "Eve" -LastName "Saunders" -JobTitle "Accounts Coordinator"
```

![New-FinanceUser.ps1 — eve.saunders created successfully] <img width="756" height="246" alt="image" src="https://github.com/user-attachments/assets/1fa67e3a-dcc0-4667-9372-1bb4d844d74d" />

**ADUC verification — all Finance users confirmed:**

![ADUC — Finance OU showing all users and Finance-Users security group] <img width="639" height="288" alt="image" src="https://github.com/user-attachments/assets/bb7de941-471a-48d7-901b-c9615c933400" />

**WS-01 login validation:**

Both users logged into WS-01 as domain users for the first time. The domain policy triggered an immediate password change prompt on first sign-in, confirming the `ChangePasswordAtLogon` flag was set correctly by the script.

![WS-01 — Lauren Esbrand password change prompt on first login] <img width="839" height="464" alt="image" src="https://github.com/user-attachments/assets/a24bd4dc-e077-49fb-ac3e-45ce3ca6bca4" />

![WS-01 — Lauren Esbrand successfully logged in] <img width="993" height="360" alt="image" src="https://github.com/user-attachments/assets/ba4dc8c7-39a8-4f56-ae5e-1689048accb7" />

![WS-01 — Eve Saunders password change prompt on first login] <img width="598" height="442" alt="image" src="https://github.com/user-attachments/assets/be03f868-da67-49d6-9f6f-62e5f7e28494" />

![WS-01 — Eve Saunders successfully logged in] <img width="980" height="295" alt="image" src="https://github.com/user-attachments/assets/7e92b8f2-08de-4729-b99a-160ae0d0aacc" />


**Ticket thread and closure:**

After validating both logins the ticket was updated with resolution notes. Katherine confirmed both users were able to access their accounts with no issues. Ticket closed by Ademola Durodola.

![Ticket thread — agent notes and resolution] <img width="965" height="683" alt="image" src="https://github.com/user-attachments/assets/d56a3362-cb0d-4be6-94fa-5b482ec6b1a7" />

![Ticket closed — Katherine confirms both users logged in successfully] <img width="999" height="613" alt="image" src="https://github.com/user-attachments/assets/8f3467a8-6fe7-4b6f-999e-4eafe1b34619" />


---
 
### T003-B — HR
 
**Submitted by:** Sandra Mills (`sandra.mills@lab.local`)
**New hires:** Adrianna Sita, Jenaiah Morris
**Script:** `New-HRUser.ps1`
 
Sandra submitted Ticket #215790 via the osTicket client portal requesting domain accounts for two incoming HR team members.
 
![Ticket #215790 — Sandra's submission and ticket details] <img width="831" height="633" alt="image" src="https://github.com/user-attachments/assets/458cba0b-d3e9-4ad8-883a-aa547f4e8fb6" />

**Script run — Adrianna Sita:**
 
```powershell
.\New-HRUser.ps1 -FirstName "Adrianna" -LastName "Sita" -JobTitle "HR Generalist"
```
 
![New-HRUser.ps1 — adrianna.sita created successfully] <img width="676" height="252" alt="image" src="https://github.com/user-attachments/assets/0afcf042-ee96-43af-ad17-ef7de7291294" />

**Script run — Jenaiah Morris:**
 
```powershell
.\New-HRUser.ps1 -FirstName "Jenaiah" -LastName "Morris" -JobTitle "Recruitment Coordinator"
```
 
![New-HRUser.ps1 — jenaiah.morris created successfully]  <img width="667" height="318" alt="image" src="https://github.com/user-attachments/assets/846effcf-9990-41f2-80bc-4900a9cedd30" />

 
**ADUC verification — all HR users confirmed:**
 
![ADUC — HR OU showing all users and HR-Users security group] <img width="655" height="231" alt="image" src="https://github.com/user-attachments/assets/9d10a34c-99a7-4345-bb0f-92c9720fc3cc" />

 
**WS-01 login validation:**
 
Both users logged into WS-01 successfully. Adrianna Sita's login was additionally verified via `whoami` in Command Prompt, confirming the domain context as `lab\adrianna.sita`.
 
![WS-01 — Adrianna Sita welcome screen] <img width="785" height="430" alt="image" src="https://github.com/user-attachments/assets/f9ee4368-2e4b-46a2-b881-962e53cfb093" />

![WS-01 — Adrianna Sita whoami confirmation] <img width="390" height="118" alt="image" src="https://github.com/user-attachments/assets/4de6b998-a083-477a-a3fe-192d559fb4bf" />

![WS-01 — Jenaiah Morris welcome screen] <img width="747" height="354" alt="image" src="https://github.com/user-attachments/assets/f4dd63c9-dfe9-4d0d-9135-46576db0db87" />

**Ticket thread and closure:**
 
After validating both logins the ticket was updated with resolution notes. Sandra confirmed both users logged in successfully with no issues. Ticket closed by Ademola Durodola.
 
![Ticket thread — agent notes, accounts created, login validation] <img width="978" height="905" alt="image" src="https://github.com/user-attachments/assets/ecf21648-3ec1-457c-8269-47ee1befd8da" />

![Ticket closed — Sandra confirms both users logged in successfully] <img width="973" height="734" alt="image" src="https://github.com/user-attachments/assets/3053f4fc-30b5-4806-8d6d-ebc029b13a01" />

---
 
### T003-C — IT
 
**Submitted by:** Paul Atreides (`paul.atreides@lab.local`)
**New hires:** Michael Cabrera, Nazeer England, Olivia Fowler
**Script:** `New-ITUser.ps1`
 
Paul submitted Ticket #986055 via the osTicket client portal requesting domain accounts for three incoming IT staff members. IT users receive an additional group assignment beyond the standard department script — all IT staff are added to `Helpdesk-Admins`, granting delegated AD control across all department OUs without Domain Admin rights.
 
![Ticket #986055 — Paul's submission and ticket details] <img width="833" height="580" alt="image" src="https://github.com/user-attachments/assets/2f7785a1-5599-41be-8850-7b0d5590bda4" />

**Script run — Michael Cabrera:**
 
```powershell
.\New-ITUser.ps1 -FirstName "Michael" -LastName "Cabrera" -JobTitle "IT Support Specialist"
```
 
![New-ITUser.ps1 — michael.cabrera created, IT-Users and Helpdesk-Admins assigned] <img width="707" height="267" alt="image" src="https://github.com/user-attachments/assets/24c42112-abf3-4a83-88b4-58a2e17f25a3" />

 
**Script run — Nazeer England:**
 
```powershell
.\New-ITUser.ps1 -FirstName "Nazeer" -LastName "England" -JobTitle "Systems Technician"
```
 
![New-ITUser.ps1 — nazeer.england created, IT-Users and Helpdesk-Admins assigned] <img width="689" height="264" alt="image" src="https://github.com/user-attachments/assets/e72332f7-d9f7-4944-a50b-231e9c4c0164" />

 
**Script run — Olivia Fowler:**
 
```powershell
.\New-ITUser.ps1 -FirstName "Olivia" -LastName "Fowler" -JobTitle "Help Desk Technician"
```
 
![New-ITUser.ps1 — olivia.fowler created, IT-Users and Helpdesk-Admins assigned] <img width="639" height="263" alt="image" src="https://github.com/user-attachments/assets/7b51d2d6-1565-446a-98fe-fc2385dc8557" />

 
Each script output confirms group membership as `Domain Users, IT-Users, Helpdesk-Admins` — verifying both assignments were applied automatically without manual intervention.
 
**ADUC verification — all IT users confirmed:**
 
![ADUC — IT OU showing all users, IT-Users and Helpdesk-Admins security groups] <img width="729" height="247" alt="image" src="https://github.com/user-attachments/assets/cc2e10e0-1b74-4e6a-a32a-843a28b5b0d6" />

 
**WS-01 login validation:**
 
Login screenshots were omitted for IT users — the pattern was established during Finance and HR onboarding. Group membership confirmation in the script output (`IT-Users, Helpdesk-Admins`) serves as the primary verification for access assignment.
 
**Ticket thread and closure:**
 
After account creation and AD verification the ticket was updated with resolution notes. Paul confirmed all three users were able to log in successfully. Ticket closed by Ademola Durodola.
 
![Ticket thread — agent notes, accounts created with Helpdesk-Admins access noted] <img width="968" height="800" alt="image" src="https://github.com/user-attachments/assets/567ef0a4-5962-41cf-9d4d-04ccf9c05c49" />

![Ticket closed — Paul confirms all three users logged in successfully] <img width="960" height="545" alt="image" src="https://github.com/user-attachments/assets/a95b2988-60dc-48ed-9302-4015f82ef722" />

 
---
### T003-D — Sales

*Pending.*

---

## Key Concepts Demonstrated

**Technical**
- Organisational Unit provisioning via PowerShell (`New-OUStructure.ps1`)
- Bulk AD user creation via scripted automation (`New-DeptLeaders.ps1`)
- Security group creation and user assignment via PowerShell (`New-SecurityGroups.ps1`)
- Department-locked user onboarding scripts (`New-FinanceUser.ps1`, `New-HRUser.ps1`, `New-ITUser.ps1`, `New-SalesUser.ps1`)
- Least privilege access model — permissions via group membership, never direct assignment
- Domain password policy enforcement and troubleshooting
- First logon password change enforcement via `ChangePasswordAtLogon` flag

**IT Support Workflow**
- Manager submits ticket on behalf of new hire — new employees have no existing system access to self-serve
- Full ticket lifecycle documented — submission, agent notes, resolution, manager confirmation, closure
- Error encountered, diagnosed, and resolved (password complexity) — documented transparently

**Security Practices**
- Temporary passwords meet domain complexity and length requirements (14 character minimum)
- All accounts set to force password change at first logon
- No cross-department share access by default
- IT helpdesk staff granted delegated AD control without Domain Admin rights

---

## Notes

- `New-OUStructure.ps1` must run before any user creation script
- `New-SecurityGroups.ps1` must run before department onboarding scripts
- Domain password policy: 14 character minimum, complexity enabled, lockout threshold 3 attempts, 30 minute duration
- Temp password used across all scripts: `TempPassword123!`
- All scripts saved to `C:\Scripts\` on DC-01 and run using `.\scriptname.ps1`
