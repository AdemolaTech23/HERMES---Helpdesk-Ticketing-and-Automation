# T005 — PowerShell Execution Policy Enforcement via Group Policy

## Overview

| Field | Details |
|---|---|
| **Type** | Security Control Implementation |
| **Policy Name** | Workstation-PS-ExecutionPolicy |
| **Scope** | All domain-joined workstations (via Departments OU) |
| **Enforced On** | WS-01 (192.168.0.70) |
| **Implemented By** | Ademola Durodola |
| **Date** | 5/1/2026 |

---

## Background & Security Rationale

PowerShell is one of the most commonly abused tools in modern attacks. Threat actors use it for payload delivery, lateral movement, credential dumping, and persistence — often by executing unsigned scripts directly on victim machines. According to MITRE ATT&CK, PowerShell abuse falls under **T1059.001 (Command and Scripting Interpreter: PowerShell)** and is consistently one of the most observed techniques in real-world intrusions.

The default execution policy on Windows workstations is `Restricted` or `RemoteSigned` depending on the build — but these are **local settings** that any user or attacker with access to the machine can change via:

```powershell
Set-ExecutionPolicy Unrestricted -Scope CurrentUser
```

A locally-configured execution policy provides no real security guarantee. The only way to enforce it reliably across an enterprise is through **Group Policy**, which applies the setting at the computer level and cannot be overridden by standard users.

This ticket documents the implementation of a GPO that enforces `AllSigned` execution policy across all domain workstations — ensuring only scripts signed by a trusted publisher can execute, regardless of what a local user attempts to configure.

---

## Policy Configuration

### Step 1 — Create the GPO

In **Group Policy Management Console (GPMC)** on DC-01:

`Group Policy Objects` → right-click → **New**

Name: `Workstation-PS-ExecutionPolicy`

![New GPO named Workstation-PS-ExecutionPolicy](assets/Group-polciy-name.png)

---

### Step 2 — Configure the Execution Policy Setting

Right-click the new GPO → **Edit**

Navigate to:
```
Computer Configuration
└── Policies
    └── Administrative Templates
        └── Windows Components
            └── Windows PowerShell
                └── Turn on Script Execution
```

Settings applied:
- **State:** Enabled
- **Execution Policy:** Allow only signed scripts

![Turn on Script Execution — Enabled, Allow only signed scripts](assets/Powershell-script-option.png)

**Why Computer Configuration and not User Configuration?**

This setting exists in both locations. Computer Configuration was chosen deliberately — it applies to the machine regardless of which user is logged in, and it takes precedence over User Configuration per Microsoft's GPO processing order. A policy set only at the user level could be bypassed by logging in as a different user. Machine-level enforcement closes that gap.

---

### Step 3 — Link the GPO

The GPO was linked to the **Departments** OU so it applies to all workstations whose computer objects fall within that OU hierarchy.

In GPMC: right-click **Departments** → **Link an Existing GPO** → select `Workstation-PS-ExecutionPolicy`

![Select GPO dialog — Workstation-PS-ExecutionPolicy selected](assets/Linked-group-policy.png)

GPO confirmed linked with Link Enabled: **Yes** and GPO Status: **Enabled**.

![GPO linked to Departments OU — confirmed enabled](assets/GPO-confirmed.png)

---

## OU Placement & Policy Inheritance

WS-01's computer object is located at:
```
lab.local → IT → Computers
```

![WS-01 computer object in IT → Computers OU](assets/Computer_location_in_AD.png)

The **IT** OU sits as a child of **lab.local**, which inherits Group Policy from parent OUs in the domain. The Departments OU link reaches WS-01 through the domain's policy inheritance chain — confirmed by `gpresult /r` showing the policy applied with the source listed as `DC01.lab.local`.

This is expected behavior. GPOs linked higher in the OU hierarchy propagate downward unless inheritance is explicitly blocked. No blocking is configured in this lab, so the policy applies cleanly.

> **Note for production environments:** In a real deployment you would scope this GPO more precisely — either by moving workstation computer objects into OUs directly under Departments, or by using **Security Group Filtering** to target only specific machines. This prevents the policy from unintentionally applying to servers or other systems in the domain.

---

## Enforcement Verification on WS-01

### gpupdate /force

After linking the GPO, `gpupdate /force` was run on WS-01 from an elevated PowerShell session to trigger immediate policy refresh:

```powershell
gpupdate /force
```

Both Computer Policy and User Policy updated successfully.

![gpupdate /force — Computer and User Policy completed successfully](assets/group-policy-update-command-ran.png)

> **Note:** Computer-side GPO changes sometimes require a full machine restart to apply. If `gpresult` does not show the policy after `gpupdate /force`, restart the workstation and re-verify.

---

### gpresult /r — Policy Confirmed Applied

```powershell
gpresult /r
```

Under **Computer Settings → Applied Group Policy Objects**, `Workstation-PS-ExecutionPolicy` is confirmed present alongside the Default Domain Policy.

![gpresult /r — Workstation-PS-ExecutionPolicy applied on WS-01](assets/GP_result_on_WS01.png)

The policy source is listed as `DC01.lab.local`, confirming it was delivered from the domain controller and is not a local setting.

---

### Enforcement Test — Unsigned Script Blocked

A test script (`Test.script.ps1`) containing a single `Write-Host` command was created on WS-01 and executed:

```powershell
.\Test.script.ps1
```

Execution was blocked with the following error:

```
File C:\Users\michael.cabrera\Documents\Test.script.ps1 cannot be loaded.
The file C:\Users\michael.cabrera\Documents\Test.script.ps1 is not digitally signed.
You cannot run this script on the current system.
CategoryInfo: SecurityError: [] PSSecurityException
FullyQualifiedErrorId: UnauthorizedAccess
```

![Unsigned script blocked — PSSecurityException UnauthorizedAccess](assets/script_not_digitally_signed_message_.png)

The policy is enforcing correctly. An unsigned script that would have executed without issue under the default execution policy is now blocked at the machine level by Group Policy.

---

## What This Protects Against

| Threat | How This Policy Mitigates It |
|---|---|
| Malicious unsigned scripts dropped via phishing | Cannot execute — no digital signature |
| Attacker changing execution policy locally | Computer-level GPO overrides local setting |
| Lateral movement via PowerShell payloads | Unsigned payloads blocked on all domain workstations |
| Insider threat running unauthorized scripts | Must be signed by a trusted publisher to execute |

This maps directly to **MITRE ATT&CK T1059.001** — adversaries using PowerShell as an execution vehicle. Enforcing `AllSigned` via GPO removes unsigned PowerShell as a viable attack vector on domain workstations.

---

## Exception Process

Legitimate IT scripts (such as those in `C:\Scripts\` on DC-01) would need to be **code-signed** with a certificate from a trusted internal Certificate Authority before they could run on workstations under this policy.

In a production environment the exception process would be:

1. IT submits a request to have a script reviewed and signed
2. A trusted team member signs the script using a code-signing certificate issued by the internal CA
3. The signed script is tested in a staging environment before deployment
4. Signed scripts are distributed via a controlled path — not copied manually to workstations

For this lab environment, administrative scripts are run directly on DC-01 where the GPO does not apply (DC-01 is in the Domain Controllers OU, not under Departments), preserving full scripting capability on the server while enforcing the restriction on workstations.

---

## Summary

| | |
|---|---|
| **Control Implemented** | PowerShell execution policy enforced via GPO — Allow only signed scripts |
| **GPO Name** | Workstation-PS-ExecutionPolicy |
| **Linked To** | Departments OU |
| **Applies To** | WS-01 (confirmed via gpresult) |
| **Enforcement Level** | Computer Configuration — overrides local user settings |
| **Verified** | Unsigned script blocked with PSSecurityException |
| **MITRE ATT&CK** | T1059.001 — Command and Scripting Interpreter: PowerShell |
| **Follow-up** | Production deployment requires internal CA and code-signing workflow |
