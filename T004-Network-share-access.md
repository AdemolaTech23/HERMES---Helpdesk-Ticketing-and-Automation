# T004 — Unable to Access IT Network Share

## Ticket Details

| Field | Details |
|---|---|
| **Ticket #** | 779507 |
| **Title** | Unable to access IT network share |
| **Submitted By** | Paul Atreides (IT Manager) — on behalf of Michael Cabrera |
| **Affected User** | michael.cabrera |
| **Workstation** | WS-01 (192.168.0.70) |
| **Priority** | High |
| **Status** | Closed |
| **Opened** | 4/27/26 11:28 PM |
| **Closed** | 4/27/26 11:32 PM |
| **Assigned To** | Ademola Durodola |

---

## User Report

Paul Atreides submitted the following via the osTicket client portal:

> "Team, Michael is unable to access the IT network drive (`\\DC-01\IT`). He's receiving a 'network path not found' error. Can you take a look and get this resolved? – Paul"

![Ticket pipeline — submission and initial notes] <img width="1012" height="873" alt="image" src="https://github.com/user-attachments/assets/385520de-8e5d-44e9-bfb5-54b44b720d0c" />


---

## Prerequisites

### Paul Atreides — Account Lockout and Disable

Before troubleshooting the share access issue, the technician needed Paul to log into WS-01 to reproduce the problem from the affected workstation. Paul was unable to log in — two separate issues were discovered.

**Issue 1 — Account Locked Out**

Paul attempted to log into WS-01 and received the following error:

> "The referenced account is currently locked out and may not be logged on to."

![Paul Atreides — account locked out error]  

The lockout threshold for the domain is set to 3 failed attempts with a 30-minute lockout duration (documented in T002). Paul had hit the threshold.

**Issue 2 — Account Disabled**

Further investigation in ADUC revealed that paul.atreides was also disabled — a separate condition from the lockout. This is a lab provisioning artifact: the account was not enabled at creation time.

![Paul Atreides — account disabled error] <img width="808" height="464" alt="image" src="https://github.com/user-attachments/assets/0aaf3166-4d31-4e2d-8773-4a9eb825fe48" />


**Resolution — Re-enable Account**

The account was re-enabled via ADUC: right-click `paul.atreides` → **Enable Account**.

![ADUC — Enable Account for paul.atreides] <img width="809" height="465" alt="image" src="https://github.com/user-attachments/assets/85892d36-2d1d-45f5-93bd-76599f465a9d" />


**Resolution — Unlock Account**

With the account re-enabled, `Unlock-ADAccount.ps1` was run against paul.atreides to clear the lockout:

```powershell
.\Unlock-ADAccount.ps1 -Username "paul.atreides"
```

Output confirmed LockedOut: True, 3 bad logons, and returned SUCCESS on unlock.

![Unlock-ADAccount.ps1 output — paul.atreides] <img width="873" height="441" alt="image" src="https://github.com/user-attachments/assets/2a53c63a-765a-48d4-a749-af2cfa32d546" />


**Verification**

Paul logged into WS-01 successfully. `whoami` confirmed `lab\paul.atreides`.

![whoami confirming lab\paul.atreides] <img width="349" height="113" alt="image" src="https://github.com/user-attachments/assets/48a3625f-6687-4c33-b69c-b1a802a0c122" />


With Paul's account restored, share troubleshooting could proceed.

---

## Troubleshooting

### Step 1 — Verify Share Availability on DC-01

Before touching the workstation, the technician verified that the shares actually exist on DC-01. This rules out a provisioning failure before any workstation-level investigation.

`Get-SmbShare` was run on DC-01:

```powershell
Get-SmbShare | Select-Object Name, Path, Description
```

All four department shares were confirmed active and pointing to the correct paths:

| Share | Path |
|---|---|
| Finance | C:\Shares\Finance |
| HR | C:\Shares\HR |
| IT | C:\Shares\IT |
| Sales | C:\Shares\Sales |

![Get-SmbShare output on DC-01] <img width="696" height="288" alt="image" src="https://github.com/user-attachments/assets/06662249-7f15-41b2-93ff-ffb740d62ce4" />


The `New-DeptShares.ps1` script run was also reviewed to confirm NTFS and SMB permissions were applied correctly at creation:

- NTFS: IT-Users (Modify), Helpdesk-Admins (ReadAndExecute), Domain Admins (Full), SYSTEM (Full)
- SMB: IT-Users (Change), Helpdesk-Admins (Read), Domain Admins (Full)
- Everyone removed from all shares

![New-DeptShares.ps1 run output] <img width="822" height="378" alt="image" src="https://github.com/user-attachments/assets/22cd7319-bece-4105-b5e2-7562d26fd7e2" />


Server-side configuration confirmed clean. Troubleshooting moved to WS-01.

---

### Step 2 — Workstation-Level Troubleshooting

The following commands were run from WS-01 as michael.cabrera to isolate the failure point.

**UNC Path Tests**

```
\\DC-01\IT          → The network path was not found.
\\192.168.0.10\IT   → The network name cannot be found.
```

Both hostname and IP-based UNC paths failed. This rules out a purely DNS-related issue — if it were only DNS, the IP path would have succeeded.

**Ping Tests**

```powershell
ping DC-01          # Ping request could not find host DC-01.
ping 192.168.0.10   # Reply from 192.168.0.10 — 4/4 packets received, 0% loss.
```

IP-level connectivity to DC-01 is confirmed. DNS hostname resolution for DC-01 is failing on WS-01.

**SMB Port Test**

```powershell
Test-NetConnection -ComputerName 192.168.0.10 -Port 445
# TcpTestSucceeded: True
```

Port 445 is open. The SMB service is reachable.

**Network Profile**

```powershell
Get-NetConnectionProfile
# NetworkCategory: DomainAuthenticated
```

WS-01 is correctly domain-joined and authenticating. Not a network category issue.

**Share Enumeration**

```powershell
net view \\192.168.0.10
```

All four shares listed successfully: Finance, HR, IT, Sales (plus NETLOGON and SYSVOL).

![net view output — all shares visible] <img width="596" height="264" alt="image" src="https://github.com/user-attachments/assets/4cb7eb59-ff92-4f28-b404-846e4e70869d" />


The shares are visible and the SMB session can be established by IP — but UNC path access is still failing.

![Full troubleshooting output — UNC failures and ping results] <img width="798" height="484" alt="image" src="https://github.com/user-attachments/assets/0544b561-ce70-49c8-ba99-7d9302d4ef40" />

**Firewall and Service Check (on DC-01)**

```powershell
Get-Service LanmanServer
Get-NetFirewallRule -Displaygroup "File and Printer Sharing" | Select-Object DisplayName, Enabled, Direction
```

LanmanServer service: **Running**. All File and Printer Sharing firewall rules: **Enabled**.

![LanmanServer service and firewall rules] <img width="943" height="511" alt="image" src="https://github.com/user-attachments/assets/1a814332-03e0-4848-8d9d-0445e05dc824" />


Server-side firewall and service configuration confirmed clean.

---

### Root Cause

WS-01 is not using DC-01 as its DNS server. Hostname resolution for `DC-01` is failing, which causes `\\DC-01\IT` to fail immediately. The IP-based path `\\192.168.0.10\IT` also fails because SMB requires a valid NetBIOS/DNS name resolution to establish a named session — the IP alone is insufficient for standard UNC access without an explicit drive mapping.

This is a DNS misconfiguration on WS-01: the NIC's preferred DNS server should point to `192.168.0.10` (DC-01), which hosts DNS for the `lab.local` domain. In a lab environment this commonly occurs when DHCP is not configured to assign the DC as the DNS server automatically.

---

## Resolution

The IT share was mapped manually using the IP address to bypass the name resolution failure:

```powershell
net use Z: \\192.168.0.10\IT
```

Drive mapped successfully. Z: drive appeared in This PC as `IT (\\192.168.0.10) (Z:)` with full capacity visible.

![Z drive mapped and accessible in This PC] <img width="782" height="528" alt="image" src="https://github.com/user-attachments/assets/57537441-9568-4b61-9c5f-bf0efcdd7e2a" />


Michael Cabrera confirmed access to the IT network share is restored.

> **Note:** The manual drive mapping is a temporary resolution. The underlying DNS misconfiguration on WS-01 should be addressed by setting the preferred DNS server on the NIC to `192.168.0.10` to prevent recurrence.

---

## Ticket Resolution & Close

The technician posted a user-facing reply to Paul Atreides:

> "Hi Paul, the issue has been resolved. Michael is now able to access the IT network drive successfully. The problem was related to a name resolution issue on the workstation, which was preventing access using the standard path. The drive has been mapped and is working as expected. Let me know if you want us to take a deeper look at the workstation configuration to prevent this going forward. – IT Support"

Final internal note documented the resolution and underlying cause. Ticket closed by Ademola Durodola.

![Ticket resolution notes and close] <img width="1019" height="781" alt="image" src="https://github.com/user-attachments/assets/3d1e336e-02f8-4fae-b512-a941b35df9cb" />

---

## Summary

| | |
|---|---|
| **Reported Issue** | michael.cabrera unable to access `\\DC-01\IT` from WS-01 |
| **Prerequisite** | paul.atreides account was locked out and disabled — re-enabled via ADUC, unlocked via Unlock-ADAccount.ps1 |
| **Root Cause** | DNS misconfiguration on WS-01 — DC-01 not set as preferred DNS server, causing hostname and UNC path resolution failure |
| **Resolution** | IT share manually mapped as Z: drive via `net use Z: \\192.168.0.10\IT` |
| **Follow-up** | NIC DNS settings on WS-01 should be updated to point to 192.168.0.10 |
| **Scripts Used** | Unlock-ADAccount.ps1 |
