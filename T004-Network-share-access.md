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

![Ticket pipeline — submission and initial notes](assets/TicketpipelineNetdrive.png)

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

![Paul Atreides — account disabled error](assets/Paul_account_disabled.png)

**Resolution — Re-enable Account**

The account was re-enabled via ADUC: right-click `paul.atreides` → **Enable Account**.

![ADUC — Enable Account for paul.atreides](assets/Paul_account_enabled.png)

**Resolution — Unlock Account**

With the account re-enabled, `Unlock-ADAccount.ps1` was run against paul.atreides to clear the lockout:

```powershell
.\Unlock-ADAccount.ps1 -Username "paul.atreides"
```

Output confirmed LockedOut: True, 3 bad logons, and returned SUCCESS on unlock.

![Unlock-ADAccount.ps1 output — paul.atreides](assets/Unlocked_paul_atreides_account.png)

**Verification**

Paul logged into WS-01 successfully. `whoami` confirmed `lab\paul.atreides`.

![whoami confirming lab\paul.atreides](assets/Paul_logged_in.png)

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

![Get-SmbShare output on DC-01](assets/Share_existence_verification.png)

The `New-DeptShares.ps1` script run was also reviewed to confirm NTFS and SMB permissions were applied correctly at creation:

- NTFS: IT-Users (Modify), Helpdesk-Admins (ReadAndExecute), Domain Admins (Full), SYSTEM (Full)
- SMB: IT-Users (Change), Helpdesk-Admins (Read), Domain Admins (Full)
- Everyone removed from all shares

![New-DeptShares.ps1 run output](assets/New-DeptShare-Script-run1.png)

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

![net view output — all shares visible](assets/list_al_directories.png)

The shares are visible and the SMB session can be established by IP — but UNC path access is still failing.

![Full troubleshooting output — UNC failures and ping results](assets/Path_cannot_be_found.png)

**Firewall and Service Check (on DC-01)**

```powershell
Get-Service LanmanServer
Get-NetFirewallRule -Displaygroup "File and Printer Sharing" | Select-Object DisplayName, Enabled, Direction
```

LanmanServer service: **Running**. All File and Printer Sharing firewall rules: **Enabled**.

![LanmanServer service and firewall rules](assets/Firewall_rule_and_service_check.png)

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

![Z drive mapped and accessible in This PC](assets/Z_drive_now_available.png)

Michael Cabrera confirmed access to the IT network share is restored.

> **Note:** The manual drive mapping is a temporary resolution. The underlying DNS misconfiguration on WS-01 should be addressed by setting the preferred DNS server on the NIC to `192.168.0.10` to prevent recurrence.

---

## Ticket Resolution & Close

The technician posted a user-facing reply to Paul Atreides:

> "Hi Paul, the issue has been resolved. Michael is now able to access the IT network drive successfully. The problem was related to a name resolution issue on the workstation, which was preventing access using the standard path. The drive has been mapped and is working as expected. Let me know if you want us to take a deeper look at the workstation configuration to prevent this going forward. – IT Support"

Final internal note documented the resolution and underlying cause. Ticket closed by Ademola Durodola.

![Ticket resolution notes and close](assets/TicketpipelineNetdrive2.png)

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
