# T002 — Account Lockout

## Prerequisites — Account Lockout Policy Configuration

Before this ticket scenario was simulated, an Account Lockout Policy was configured on DC-01 via Group Policy Management to reflect a realistic corporate security baseline.

**Policy applied to:** Default Domain Policy — `DC01.LAB.LOCAL`

**Settings configured:**

| Policy | Value |
|---|---|
| Account lockout threshold | 3 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset account lockout counter after | 30 minutes |

This policy ensures that accounts are automatically locked after repeated failed attempts, protecting against brute force and credential stuffing attacks.

![Group Policy lockout configuration] <img width="1007" height="607" alt="image" src="https://github.com/user-attachments/assets/a5a72248-e2cf-4ca4-87ee-c463b1d34a79" />


---

## Ticket Details

| Field | Details |
|---|---|
| Ticket ID | #170269 |
| Date | 2026-04-16 |
| Submitted via | osTicket (web portal) |
| Priority | Normal (downgraded from High) |
| Status | Closed |
| Assigned To | Ademola Durodola |
| Environment | DC-01 (Windows Server 2022) · WS-01 (Windows 10 Pro) · INFRA-01 (osTicket) |

---

## User Report

User **Jasmine Rodgers** (`jasmine.rodgers@lablocal.com`) submitted a ticket from WS-01 after being locked out of her domain account following multiple failed login attempts.

> "Hi, I tried logging into my computer this morning but kept getting my password wrong. Now it won't let me in at all and says my account is locked out. Can you please unlock it so I can get back to work?"

![Jasmine submits lockout ticket from WS-01]  <img width="933" height="515" alt="image" src="https://github.com/user-attachments/assets/3e096fc1-3a03-47bd-827a-e607dab432ad" />


---

## Step 1 — Ticket Intake and Triage

Ticket located in the osTicket Staff Control Panel. Assigned to Ademola Durodola. Priority adjusted from **High → Normal**.

**Reason for downgrade:** Single user affected, no system-wide impact. User can continue other tasks while account is restored.

![Technician reviews ticket]  <img width="989" height="628" alt="image" src="https://github.com/user-attachments/assets/b3aa9a54-020d-4ee3-8b0f-1cfaa7957d18" />


---

## Step 2 — Internal Analysis and Verification

Before taking action, an internal note was added documenting the approach:

> "User reports account lockout after multiple failed login attempts. Verified account status in Active Directory and confirmed account is locked. Proceeding to unlock account using administrative tools."

Account status verified on DC-01 via PowerShell:

```powershell
Get-ADUser -Identity "jasmine.rodgers" -Properties LockedOut | Select Name, LockedOut
```

Output confirmed `LockedOut : True`.

![PowerShell lockout verification] <img width="948" height="134" alt="image" src="https://github.com/user-attachments/assets/1df69f53-90b3-459e-ba68-718b520eff1b" />

![WS-01 lockout screen]  <img width="861" height="491" alt="image" src="https://github.com/user-attachments/assets/cd0635aa-8da5-4962-9c3f-d725a066cc73" />

---

## Step 3 — Remediation

Account unlocked on DC-01 using `Unlock-ADAccount.ps1`:

```powershell
# Unlock-ADAccount.ps1
# HERMES — Helpdesk Ticketing and Automation
# Checks if a domain user account is locked and unlocks it if so.
# Run on DC-01 with Domain Admin or Account Operator privileges.
#
# Usage:
#   .\Unlock-ADAccount.ps1 -Username "ADusername"
#
# Example:
#   .\Unlock-ADAccount.ps1 -Username "john.smith"

param (
    [Parameter(Mandatory = $true)]
    [string]$Username
)

Import-Module ActiveDirectory

# Check the account exists
try {
    $user = Get-ADUser -Identity $Username -Properties LockedOut, LastLogonDate, BadLogonCount
} catch {
    Write-Host "ERROR: User '$Username' not found in Active Directory." -ForegroundColor Red
    exit 1
}

Write-Host ""
Write-Host "Account Details" -ForegroundColor Cyan
Write-Host "---------------"
Write-Host "Name         : $($user.Name)"
Write-Host "Username     : $($user.SamAccountName)"
Write-Host "Locked Out   : $($user.LockedOut)"
Write-Host "Bad Logons   : $($user.BadLogonCount)"
Write-Host "Last Logon   : $($user.LastLogonDate)"
Write-Host ""

if ($user.LockedOut) {
    Write-Host "Account is locked. Unlocking..." -ForegroundColor Yellow
    try {
        Unlock-ADAccount -Identity $Username
        Write-Host "SUCCESS: Account '$Username' has been unlocked." -ForegroundColor Green
        Write-Host ""
        Write-Host "Next steps:" -ForegroundColor Cyan
        Write-Host "  - Communicate to user that account is unlocked"
        Write-Host "  - Advise user to log in and verify access"
        Write-Host "  - If repeated lockouts occur, investigate source (bad saved credentials, mapped drives, mobile device)"
    } catch {
        Write-Host "ERROR: Failed to unlock account. $_" -ForegroundColor Red
    }
} else {
    Write-Host "Account is NOT locked." -ForegroundColor Green
    Write-Host "If the user still cannot log in, the issue may be an incorrect password."
    Write-Host "Consider running Reset-UserPassword.ps1 instead."
}

Write-Host ""
```

Run command used:

```powershell
.\Unlock-ADAccount.ps1 -Username "jasmine.rodgers"
```

Script output confirmed:
- Account Details displayed: Name, Username, Locked Out status, Bad Logon count, Last Logon
- Bad Logons: 3 (matching the lockout threshold set in Group Policy)
- Account unlocked successfully

![Script ran successfully] <img width="894" height="351" alt="image" src="https://github.com/user-attachments/assets/0660144a-d3a1-442e-bc14-bc9f69d8c0ed" />


Internal note added to ticket:

> "Account unlocked in Active Directory using PowerShell script (Unlock-ADAccount.ps1). Lockout status cleared successfully."

---

## Step 4 — User Communication

Response sent to Jasmine via osTicket:

> "Hello Jasmine, your account has been unlocked. Please try logging in again using your existing password. If you continue to experience any issues, please let us know. Thank you, IT Support"

![Resolution thread]  <img width="965" height="769" alt="image" src="https://github.com/user-attachments/assets/d4118d9a-41b8-4dfb-ad35-e368845d5616" />


---

## Step 5 — User Confirmation and Closure

Jasmine replied directly in the ticket confirming resolution:

> "Hi, I'm able to log in now, and everything is working fine. Thank you for the help! — Jasmine"

Ticket reopened briefly by Jasmine's reply, then closed by technician with a final internal note:

> "Ticket closed after user confirmed successful login via ticket response."

Ticket status set to **Closed**.

![Full resolution thread and closure]  <img width="1075" height="866" alt="image" src="https://github.com/user-attachments/assets/4d7267ac-17de-4452-be21-9a28f5db6bc6" />


---

## Resolution Summary

Account lockout caused by 3 consecutive failed login attempts, triggering the domain lockout policy (threshold: 3 attempts). Account unlocked via `Unlock-ADAccount.ps1` on DC-01. User confirmed successful login via ticket reply. Ticket closed.

---

## Key Concepts Demonstrated

**Technical**
- Account lockout identification via PowerShell (`Get-ADUser`)
- Account unlock via custom PowerShell script (`Unlock-ADAccount.ps1`)
- Group Policy lockout threshold enforcement (3 invalid attempts)

**IT Support Workflow**
- Full ticket lifecycle: submission → triage → analysis → remediation → communication → user confirmation → closure
- Internal notes maintained at each action for audit trail
- Priority downgrade with documented reasoning

**Security Practices**
- Verified lockout status before taking action — did not assume
- Documented remediation method in ticket (script name referenced)
- Bad logon count noted — repeated lockouts would trigger further investigation into source (saved credentials, mapped drives, mobile device sync)

---

## Notes

- If Jasmine experiences repeated lockouts, investigate for saved credentials on WS-01 or a mobile device still using the old password
- Group Policy lockout policy documented: 3 invalid attempts, 30 minute duration

