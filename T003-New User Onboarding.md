# T003 — New User Onboarding

## Ticket Details

| Field | Details |
|---|---|
| Date | 2026-04-16 |
| Submitted via | osTicket (web portal) |
| Priority | Normal |
| Status | In Progress |
| Assigned To | Ademola Durodola |
| Environment | DC-01 (Windows Server 2022) · WS-01 (Windows 10 Pro) · INFRA-01 (osTicket) |

---

## Overview

New user onboarding is one of the most common Tier 1 helpdesk tasks. A department manager submits a ticket requesting a domain account for a newly hired employee. The technician creates the account in Active Directory, places it in the correct department OU, sets a temporary password, and confirms access before closing the ticket.

This document covers the full onboarding workflow including the AD environment setup that makes it possible.

---

## Prerequisites — Part 1: OU Structure

Before any user accounts can be created in the correct department, a standardised Organisational Unit structure was provisioned on DC-01 using `New-OUStructure.ps1`.

**Script run on DC-01:**

```powershell
.\New-OUStructure.ps1
```

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

Script output confirmed Finance OU already existed and was skipped. IT, HR, and Sales OUs were created fresh with their Users sub-OUs.

![New-OUStructure script output — start] <img width="954" height="479" alt="image" src="https://github.com/user-attachments/assets/0bd2cceb-b74a-4e84-a165-a0549640bc37" />



![New-OUStructure script output — departments created]  <img width="961" height="482" alt="image" src="https://github.com/user-attachments/assets/6d653838-a1be-4bf5-ae44-e21db6142cb9" />


![New-OUStructure script output — complete] <img width="903" height="398" alt="image" src="https://github.com/user-attachments/assets/6c7ee02c-8f83-44ab-abcf-82a400fae758" />



![OU structure verified in ADUC]  <img width="662" height="299" alt="image" src="https://github.com/user-attachments/assets/4dcddf7c-56e1-4cf8-aea3-cc89d71e7d25" />


---

## Prerequisites — Part 2: Department Leaders

With the OU structure in place, department head accounts were created using `New-DeptLeaders.ps1`. These accounts represent the managers who submit onboarding requests via osTicket.

**Script run on DC-01:**

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

**Note:** The first run failed with a password complexity error — the initial temp password `TempPass123!` did not meet the domain minimum length of 14 characters. The script was corrected and rerun with `TempPassword123!` (16 characters), which met the policy.

![New-DeptLeaders script — first run start] <img width="957" height="446" alt="image" src="https://github.com/user-attachments/assets/e13c852d-0572-4bff-96ef-6f7516d04252" />


![New-DeptLeaders script — password error on first run]  <img width="952" height="483" alt="image" src="https://github.com/user-attachments/assets/603757e7-332b-467c-952c-f5d1842ad897" />


![New-DeptLeaders script — second run]  <img width="963" height="473" alt="image" src="https://github.com/user-attachments/assets/2757ee8f-b761-42d1-9c44-d4263d9f5b27" />

![New-DeptLeaders script — accounts created]  <img width="954" height="509" alt="image" src="https://github.com/user-attachments/assets/18d765d4-0c4f-4c2a-b03f-21e2d408329d" />

![New-DeptLeaders script — summary output]  <img width="932" height="481" alt="image" src="https://github.com/user-attachments/assets/88f4b351-8126-46b3-a1a8-554a3d8d01a0" />

**ADUC verification — all four leaders in correct OUs:**

![Finance OU — Katherine Stetson] <img width="749" height="182" alt="image" src="https://github.com/user-attachments/assets/627d7be8-abcd-4551-8c7b-13d2cfd748a7" />

![HR OU — Sandra Mills]  <img width="752" height="219" alt="image" src="https://github.com/user-attachments/assets/07bdbb64-99b7-494d-8157-d6ca0bca22e1" />

![IT OU — Paul Atreides]  <img width="749" height="279" alt="image" src="https://github.com/user-attachments/assets/35e3876d-ecd9-473e-956e-93be59a7f098" />

![Sales OU — Anakin Skywalker] <img width="691" height="334" alt="image" src="https://github.com/user-attachments/assets/8696ed8d-e30c-4b3d-89b5-b8fdef37a285" />


**Passwords reset to meet domain policy:**

After the first run failure, all four accounts had their passwords reset via a bulk PowerShell command to ensure compliance with the 14 character minimum:

```powershell
$newPassword = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force

$users = @("katherine.stetson", "sandra.mills", "paul.atreides", "anakin.skywalker")

foreach ($user in $users) {
    Set-ADAccountPassword -Identity $user -NewPassword $newPassword -Reset
    Set-ADUser -Identity $user -ChangePasswordAtLogon $true
    Write-Host "Password updated: $user" -ForegroundColor Green
}
```

![Bulk password reset output] 

<img width="697" height="291" alt="image" src="https://github.com/user-attachments/assets/4df58842-eec1-43c9-83a8-a42dd0bffa77" />

<img width="798" height="375" alt="image" src="https://github.com/user-attachments/assets/0db1c340-2bcd-4f2d-a6cb-360161cc45c0" />

---

## Ticket Simulation

Each department manager will submit a separate osTicket request for their new employee. The technician handles all four using department-specific PowerShell scripts.

*Ticket simulations and script runs in progress — documentation to follow.*

---

## Key Concepts Demonstrated

**Technical**
- Organisational Unit provisioning via PowerShell (`New-OUStructure.ps1`)
- Bulk AD user creation via scripted automation (`New-DeptLeaders.ps1`)
- Domain password policy enforcement and troubleshooting
- Department-based OU placement for Group Policy application

**IT Support Workflow**
- Environment setup completed before ticket simulation — proactive sysadmin thinking
- Error encountered, diagnosed, and resolved (password complexity) — documented transparently
- Bulk password remediation via inline PowerShell — efficient, repeatable

**Security Practices**
- Temporary passwords meet domain complexity and length requirements
- All accounts set to force password change at first logon
- Password never communicated via ticket

---

## Notes

- `New-OUStructure.ps1` must always run before `New-DeptLeaders.ps1` or any user creation script
- Domain password policy: 14 character minimum, complexity enabled, lockout threshold 3 attempts
- Temp password used across all scripts: `TempPassword123!`
