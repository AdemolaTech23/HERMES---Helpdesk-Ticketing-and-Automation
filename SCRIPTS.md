# SCRIPTS — PowerShell Reference

All scripts are stored in `C:\Scripts\` on DC-01 and run with Domain Admin privileges. Full line-by-line explanations are embedded in each ticket document. This file is a single reference point for all scripts used across the HERMES repo.

---

## Table of Contents

| Script | Used In | Purpose |
|---|---|---|
| [Unlock-ADAccount.ps1](#unlock-adaccountps1) | T002, T004 | Checks lockout status and unlocks a domain user account |
| [New-OUStructure.ps1](#new-oustructureps1) | T003 | Provisions the Departments OU and all four department sub-OUs |
| [New-DeptLeaders.ps1](#new-deptleadersps1) | T003 | Creates the four department manager accounts |
| [New-SecurityGroups.ps1](#new-securitygroupsps1) | T003 | Creates the six security groups and seeds group membership |
| [New-FinanceUser.ps1](#new-financeuserps1) | T003 | Creates a Finance user, assigns to `Finance-Users` |
| [New-HRUser.ps1](#new-hruserps1) | T003 | Creates an HR user, assigns to `HR-Users` |
| [New-ITUser.ps1](#new-ituserps1) | T003 | Creates an IT user, assigns to `IT-Users` and `Helpdesk-Admins` |
| [New-SalesUser.ps1](#new-salesuserps1) | T003 | Creates a Sales user, assigns to `Sales-Users` |
| [New-DeptShares.ps1](#new-deptsharesps1) | T004 | Creates the four department SMB shares with NTFS and share permissions |

---

## Unlock-ADAccount.ps1

Checks whether a domain user account is locked out and unlocks it if so. Displays account details and bad logon count before taking action. Prints next-step guidance on completion.

**Usage:**
```powershell
.\Unlock-ADAccount.ps1 -Username "jasmine.rodgers"
```

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

---

## New-OUStructure.ps1

Creates the Departments OU and all four department sub-OUs with Users containers. Skips any OU that already exists — safe to rerun.

**Usage:**
```powershell
.\New-OUStructure.ps1
```

```powershell
# New-OUStructure.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates a standardised Organisational Unit structure in Active Directory.
# Run once during environment setup or when adding new departments.
# Run on DC-01 with Domain Admin privileges.
#
# Usage:
#   .\New-OUStructure.ps1
#
# Structure created:
#   lab.local
#   └── Departments
#       ├── Finance
#       │   └── Users
#       ├── IT
#       │   └── Users
#       ├── HR
#       │   └── Users
#       └── Sales
#           └── Users

Import-Module ActiveDirectory

# Domain root
$domainRoot = "DC=lab,DC=local"

# Departments to create
$departments = @("Finance", "IT", "HR", "Sales")

Write-Host ""
Write-Host "OU Structure Setup" -ForegroundColor Cyan
Write-Host "------------------"
Write-Host "Domain: $domainRoot"
Write-Host ""

# Create top-level Departments OU if it doesn't exist
$departmentsOU = "OU=Departments,$domainRoot"

try {
    Get-ADOrganizationalUnit -Identity $departmentsOU | Out-Null
    Write-Host "Departments OU already exists — skipping." -ForegroundColor Yellow
} catch {
    New-ADOrganizationalUnit -Name "Departments" -Path $domainRoot -ProtectedFromAccidentalDeletion $false
    Write-Host "Created: Departments OU" -ForegroundColor Green
}

Write-Host ""

# Create each department OU and its Users sub-OU
foreach ($dept in $departments) {

    $deptOU = "OU=$dept,$departmentsOU"
    $usersOU = "OU=Users,$deptOU"

    # Create department OU
    try {
        Get-ADOrganizationalUnit -Identity $deptOU | Out-Null
        Write-Host "$dept OU already exists — skipping." -ForegroundColor Yellow
    } catch {
        New-ADOrganizationalUnit -Name $dept -Path $departmentsOU -ProtectedFromAccidentalDeletion $false
        Write-Host "Created: $dept OU" -ForegroundColor Green
    }

    # Create Users sub-OU inside department
    try {
        Get-ADOrganizationalUnit -Identity $usersOU | Out-Null
        Write-Host "$dept\Users OU already exists — skipping." -ForegroundColor Yellow
    } catch {
        New-ADOrganizationalUnit -Name "Users" -Path $deptOU -ProtectedFromAccidentalDeletion $false
        Write-Host "Created: $dept\Users OU" -ForegroundColor Green
    }

    Write-Host ""
}

Write-Host "OU structure setup complete." -ForegroundColor Cyan
Write-Host ""
Write-Host "Next steps:" -ForegroundColor Cyan
Write-Host "  - Verify structure in Active Directory Users and Computers"
Write-Host "  - Run New-ADUser.ps1 to create users in the appropriate OUs"
Write-Host ""
```

---

## New-DeptLeaders.ps1

Creates the four department manager accounts in their correct department OUs. Sets a temp password with forced change at first logon. Skips any account that already exists.

> **Note:** The first run used `TempPass123!` which failed the 14-character domain minimum. The password was corrected to `TempPassword123!` and the script was rerun successfully. See [T003](T003-New%20User%20Onboarding.md) for the full error and resolution.

**Usage:**
```powershell
.\New-DeptLeaders.ps1
```

```powershell
# New-DeptLeaders.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates all department head accounts in Active Directory.
# Run once during environment setup before any ticket simulations.
# Run on DC-01 with Domain Admin privileges.
#
# Usage:
#   .\New-DeptLeaders.ps1
#
# Accounts created:
#   Katherine Stetson  — Finance Manager  — OU=Users,OU=Finance,OU=Departments,DC=lab,DC=local
#   Sandra Mills       — HR Manager       — OU=Users,OU=HR,OU=Departments,DC=lab,DC=local
#   Paul Atreides      — IT Manager       — OU=Users,OU=IT,OU=Departments,DC=lab,DC=local
#   Anakin Skywalker   — Sales Manager    — OU=Users,OU=Sales,OU=Departments,DC=lab,DC=local

Import-Module ActiveDirectory

$domainRoot   = "DC=lab,DC=local"
$tempPassword = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force

# Department leaders
$leaders = @(
   @{ FirstName = "Katherine"; LastName = "Stetson";   Department = "Finance"; Title = "Finance Manager" },
   @{ FirstName = "Sandra";    LastName = "Mills";      Department = "HR";      Title = "HR Manager"      },
   @{ FirstName = "Paul";      LastName = "Atreides";   Department = "IT";      Title = "IT Manager"      },
   @{ FirstName = "Anakin";    LastName = "Skywalker";  Department = "Sales";   Title = "Sales Manager"   }
)

Write-Host ""
Write-Host "Department Leaders Setup" -ForegroundColor Cyan
Write-Host "------------------------"
Write-Host "Domain: $domainRoot"
Write-Host ""

foreach ($leader in $leaders) {

   $username    = "$($leader.FirstName.ToLower()).$($leader.LastName.ToLower())"
   $displayName = "$($leader.FirstName) $($leader.LastName)"
   $ouPath      = "OU=Users,OU=$($leader.Department),OU=Departments,$domainRoot"
   $upn         = "$username@lab.local"

   Write-Host "Creating: $displayName — $($leader.Title)" -ForegroundColor Cyan

   # Check if user already exists
   try {
       Get-ADUser -Identity $username | Out-Null
       Write-Host "  SKIPPED: '$username' already exists in Active Directory." -ForegroundColor Yellow
       Write-Host ""
       continue
   } catch {}

   # Verify target OU exists
   try {
       Get-ADOrganizationalUnit -Identity $ouPath | Out-Null
   } catch {
       Write-Host "  ERROR: Target OU not found: $ouPath" -ForegroundColor Red
       Write-Host "  Run New-OUStructure.ps1 first." -ForegroundColor Yellow
       Write-Host ""
       continue
   }

   # Create the account
   try {
       New-ADUser `
           -Name $displayName `
           -GivenName $leader.FirstName `
           -Surname $leader.LastName `
           -SamAccountName $username `
           -UserPrincipalName $upn `
           -Path $ouPath `
           -Department $leader.Department `
           -Title $leader.Title `
           -AccountPassword $tempPassword `
           -ChangePasswordAtLogon $true `
           -Enabled $true

       Write-Host "  SUCCESS: '$username' created in $($leader.Department) OU." -ForegroundColor Green
       Write-Host "  UPN           : $upn"
       Write-Host "  Temp Password : TempPassword123!"
       Write-Host ""

   } catch {
       Write-Host "  ERROR: Failed to create '$username'. $_" -ForegroundColor Red
       Write-Host ""
   }
}

Write-Host "Department leaders setup complete." -ForegroundColor Cyan
Write-Host ""
Write-Host "Next steps:" -ForegroundColor Cyan
Write-Host "  - Verify accounts in Active Directory Users and Computers"
Write-Host "  - Communicate temporary credentials to each manager"
Write-Host "  - Run department-specific scripts for standard user creation"
Write-Host ""
```

---

## New-SecurityGroups.ps1

Creates all six department and helpdesk security groups. Places each group in the correct OU. Seeds group membership for existing users on creation. Outputs a created/skipped/failed summary on completion.

**Usage:**
```powershell
.\New-SecurityGroups.ps1
```

```powershell
# New-SecurityGroups.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates all department security groups and the Helpdesk-Admins group in AD.
# Groups are placed in their respective department OUs.
# Run on DC-01 with Domain Admin privileges.

# --- Configuration ---
$DomainRoot = "DC=lab,DC=local"

$Groups = @(
    @{
        Name        = "Finance-Users"
        Path        = "OU=Users,OU=Finance,OU=Departments,$DomainRoot"
        Description = "Finance department users - grants Read/Write access to \\DC-01\Shares\Finance"
        Scope       = "Global"
    },
    @{
        Name        = "HR-Users"
        Path        = "OU=Users,OU=HR,OU=Departments,$DomainRoot"
        Description = "HR department users - grants Read/Write access to \\DC-01\Shares\HR"
        Scope       = "Global"
    },
    @{
        Name        = "IT-Users"
        Path        = "OU=Users,OU=IT,OU=Departments,$DomainRoot"
        Description = "IT department users - grants Read/Write access to \\DC-01\Shares\IT"
        Scope       = "Global"
    },
    @{
        Name        = "Sales-Users"
        Path        = "OU=Users,OU=Sales,OU=Departments,$DomainRoot"
        Description = "Sales department users - grants Read/Write access to \\DC-01\Shares\Sales"
        Scope       = "Global"
    },
    @{
        Name        = "Dept-Managers"
        Path        = "OU=Departments,$DomainRoot"
        Description = "Department managers - scoped AD permissions over their own department OU"
        Scope       = "Global"
    },
    @{
        Name        = "Helpdesk-Admins"
        Path        = "OU=Users,OU=IT,OU=Departments,$DomainRoot"
        Description = "IT helpdesk staff - delegated AD control across all department OUs, Read on all shares"
        Scope       = "Global"
    }
)

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES — Security Group Setup" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

$Created = 0
$Skipped = 0
$Failed  = 0

foreach ($Group in $Groups) {

    Write-Host "Processing: $($Group.Name)" -ForegroundColor Yellow

    # Check if group already exists
    try {
        Get-ADGroup -Identity $Group.Name -ErrorAction Stop | Out-Null
        Write-Host "  SKIPPED: '$($Group.Name)' already exists." -ForegroundColor Yellow
        $Skipped++
        Write-Host ""
        continue
    } catch {}

    # Verify target OU exists
    try {
        Get-ADOrganizationalUnit -Identity $Group.Path -ErrorAction Stop | Out-Null
    } catch {
        Write-Host "  ERROR: Target OU not found: $($Group.Path)" -ForegroundColor Red
        Write-Host "  Run New-OUStructure.ps1 first to create the OU structure." -ForegroundColor Red
        $Failed++
        Write-Host ""
        continue
    }

    # Create the group
    try {
        New-ADGroup `
            -Name $Group.Name `
            -SamAccountName $Group.Name `
            -GroupScope $Group.Scope `
            -GroupCategory Security `
            -Path $Group.Path `
            -Description $Group.Description `
            -ErrorAction Stop

        Write-Host "  OK: '$($Group.Name)' created successfully." -ForegroundColor Green
        $Created++
    } catch {
        Write-Host "  ERROR: Failed to create '$($Group.Name)'." -ForegroundColor Red
        Write-Host "  $($_.Exception.Message)" -ForegroundColor Red
        $Failed++
    }

    Write-Host ""
}

# Add department managers to Dept-Managers group
Write-Host "Adding department managers to 'Dept-Managers' group..." -ForegroundColor Yellow
Write-Host ""

$Managers = @("katherine.stetson", "sandra.mills", "paul.atreides", "anakin.skywalker")

foreach ($Manager in $Managers) {
    try {
        Add-ADGroupMember -Identity "Dept-Managers" -Members $Manager -ErrorAction Stop
        Write-Host "  OK: '$Manager' added to 'Dept-Managers'." -ForegroundColor Green
    } catch {
        Write-Host "  WARNING: Could not add '$Manager' to 'Dept-Managers'." -ForegroundColor Yellow
        Write-Host "  $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

Write-Host ""

# Add IT manager to Helpdesk-Admins group
Write-Host "Adding IT manager to 'Helpdesk-Admins' group..." -ForegroundColor Yellow
Write-Host ""

try {
    Add-ADGroupMember -Identity "Helpdesk-Admins" -Members "paul.atreides" -ErrorAction Stop
    Write-Host "  OK: 'paul.atreides' added to 'Helpdesk-Admins'." -ForegroundColor Green
} catch {
    Write-Host "  WARNING: Could not add 'paul.atreides' to 'Helpdesk-Admins'." -ForegroundColor Yellow
    Write-Host "  $($_.Exception.Message)" -ForegroundColor Yellow
}

Write-Host ""

# Add existing Finance users to Finance-Users group
Write-Host "Adding existing Finance users to 'Finance-Users' group..." -ForegroundColor Yellow
Write-Host ""

$ExistingFinanceUsers = @("jasmine.rodgers", "katherine.stetson")

foreach ($User in $ExistingFinanceUsers) {
    try {
        Add-ADGroupMember -Identity "Finance-Users" -Members $User -ErrorAction Stop
        Write-Host "  OK: '$User' added to 'Finance-Users'." -ForegroundColor Green
    } catch {
        Write-Host "  WARNING: Could not add '$User' to 'Finance-Users'." -ForegroundColor Yellow
        Write-Host "  $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

Write-Host ""

# Summary
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Security Group Setup Complete" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "  Groups Created : $Created" -ForegroundColor Green
Write-Host "  Groups Skipped : $Skipped" -ForegroundColor Yellow
Write-Host "  Groups Failed  : $Failed" -ForegroundColor $(if ($Failed -gt 0) { "Red" } else { "Green" })
Write-Host ""
Write-Host "Next steps:" -ForegroundColor Cyan
Write-Host "  - Verify groups in Active Directory Users and Computers"
Write-Host "  - Run New-FinanceUser.ps1 to onboard Finance department users"
Write-Host "  - Run New-HRUser.ps1, New-ITUser.ps1, New-SalesUser.ps1 for other departments"
Write-Host ""
```

---

## New-FinanceUser.ps1

Creates a Finance department user. Places the user in `Finance\Users`, assigns to `Finance-Users`, sets temp password with forced change at first logon. Verifies the account after creation and prints a summary.

**Usage:**
```powershell
.\New-FinanceUser.ps1 -FirstName "Lauren" -LastName "Esbrand" -JobTitle "Finance Analyst"
```

```powershell
# New-FinanceUser.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates a new Active Directory user account locked to the Finance department.
# Assigns to Finance-Users security group, sets temp password, forces password
# change at first logon, and populates all standard AD attributes.
# Run on DC-01 with Domain Admin privileges.

param (
    [Parameter(Mandatory = $true)]
    [string]$FirstName,

    [Parameter(Mandatory = $true)]
    [string]$LastName,

    [Parameter(Mandatory = $true)]
    [string]$JobTitle
)

# --- Configuration ---
$Department    = "Finance"
$Manager       = "katherine.stetson"
$TargetOU      = "OU=Users,OU=Finance,OU=Departments,DC=lab,DC=local"
$SecurityGroup = "Finance-Users"
$TempPassword  = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force
$Company       = "lab.local"

# --- Derived Values ---
$Username    = "$($FirstName.ToLower()).$($LastName.ToLower())"
$DisplayName = "$FirstName $LastName"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES — Finance User Onboarding" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Name       : $DisplayName"
Write-Host "Username   : $Username"
Write-Host "Title      : $JobTitle"
Write-Host "Department : $Department"
Write-Host "OU         : $TargetOU"
Write-Host "Group      : $SecurityGroup"
Write-Host ""

# Step 1: Check if user already exists
Write-Host "Step 1 — Checking if user already exists..." -ForegroundColor Yellow

try {
    Get-ADUser -Identity $Username -ErrorAction Stop | Out-Null
    Write-Host "ERROR: User '$Username' already exists in Active Directory." -ForegroundColor Red
    exit 1
} catch {
    Write-Host "OK: Username '$Username' is available." -ForegroundColor Green
}

Write-Host ""

# Step 2: Verify target OU exists
Write-Host "Step 2 — Verifying target OU exists..." -ForegroundColor Yellow

try {
    Get-ADOrganizationalUnit -Identity $TargetOU -ErrorAction Stop | Out-Null
    Write-Host "OK: Target OU found." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Target OU not found: $TargetOU" -ForegroundColor Red
    Write-Host "Run New-OUStructure.ps1 first to create the OU structure." -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 3: Create the user account
Write-Host "Step 3 — Creating user account..." -ForegroundColor Yellow

try {
    New-ADUser `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@lab.local" `
        -GivenName $FirstName `
        -Surname $LastName `
        -DisplayName $DisplayName `
        -Name $DisplayName `
        -Title $JobTitle `
        -Department $Department `
        -Company $Company `
        -Manager $Manager `
        -Path $TargetOU `
        -AccountPassword $TempPassword `
        -ChangePasswordAtLogon $true `
        -Enabled $true `
        -ErrorAction Stop

    Write-Host "OK: User account '$Username' created successfully." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Failed to create user account." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 4: Add to Finance-Users security group
Write-Host "Step 4 — Adding to '$SecurityGroup' security group..." -ForegroundColor Yellow

try {
    Add-ADGroupMember -Identity $SecurityGroup -Members $Username -ErrorAction Stop
    Write-Host "OK: '$Username' added to '$SecurityGroup'." -ForegroundColor Green
} catch {
    Write-Host "WARNING: Could not add user to '$SecurityGroup'." -ForegroundColor Yellow
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}

Write-Host ""

# Step 5: Verify and summarise
Write-Host "Step 5 — Verifying account..." -ForegroundColor Yellow

$NewUser = Get-ADUser -Identity $Username -Properties DisplayName, Title, Department, DistinguishedName, Enabled

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Account Created Successfully" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Display Name : $($NewUser.DisplayName)"
Write-Host "Username     : $($NewUser.SamAccountName)"
Write-Host "Title        : $($NewUser.Title)"
Write-Host "Department   : $($NewUser.Department)"
Write-Host "Enabled      : $($NewUser.Enabled)"
Write-Host "OU Path      : $($NewUser.DistinguishedName)"
Write-Host ""
Write-Host "Temp Password   : TempPassword123!" -ForegroundColor Yellow
Write-Host "Password Change : Required at first logon" -ForegroundColor Yellow
Write-Host ""
Write-Host "Next step: Have the user log into WS-01 to verify access." -ForegroundColor Cyan
Write-Host ""
```

---

## New-HRUser.ps1

Creates an HR department user. Places the user in `HR\Users`, assigns to `HR-Users`, sets temp password with forced change at first logon.

**Usage:**
```powershell
.\New-HRUser.ps1 -FirstName "Adrianna" -LastName "Sita" -JobTitle "HR Generalist"
```

```powershell
# New-HRUser.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates a new Active Directory user account locked to the HR department.
# Assigns to HR-Users security group, sets temp password, forces password
# change at first logon, and populates all standard AD attributes.
# Run on DC-01 with Domain Admin privileges.

param (
    [Parameter(Mandatory = $true)]
    [string]$FirstName,

    [Parameter(Mandatory = $true)]
    [string]$LastName,

    [Parameter(Mandatory = $true)]
    [string]$JobTitle
)

# --- Configuration ---
$Department    = "HR"
$Manager       = "sandra.mills"
$TargetOU      = "OU=Users,OU=HR,OU=Departments,DC=lab,DC=local"
$SecurityGroup = "HR-Users"
$TempPassword  = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force
$Company       = "lab.local"

# --- Derived Values ---
$Username    = "$($FirstName.ToLower()).$($LastName.ToLower())"
$DisplayName = "$FirstName $LastName"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES - HR User Onboarding" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Name       : $DisplayName"
Write-Host "Username   : $Username"
Write-Host "Title      : $JobTitle"
Write-Host "Department : $Department"
Write-Host "OU         : $TargetOU"
Write-Host "Group      : $SecurityGroup"
Write-Host ""

# Step 1: Check if user already exists
Write-Host "Step 1 - Checking if user already exists..." -ForegroundColor Yellow

try {
    Get-ADUser -Identity $Username -ErrorAction Stop | Out-Null
    Write-Host "ERROR: User '$Username' already exists in Active Directory." -ForegroundColor Red
    exit 1
} catch {
    Write-Host "OK: Username '$Username' is available." -ForegroundColor Green
}

Write-Host ""

# Step 2: Verify target OU exists
Write-Host "Step 2 - Verifying target OU exists..." -ForegroundColor Yellow

try {
    Get-ADOrganizationalUnit -Identity $TargetOU -ErrorAction Stop | Out-Null
    Write-Host "OK: Target OU found." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Target OU not found: $TargetOU" -ForegroundColor Red
    Write-Host "Run New-OUStructure.ps1 first to create the OU structure." -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 3: Create the user account
Write-Host "Step 3 - Creating user account..." -ForegroundColor Yellow

try {
    New-ADUser `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@lab.local" `
        -GivenName $FirstName `
        -Surname $LastName `
        -DisplayName $DisplayName `
        -Name $DisplayName `
        -Title $JobTitle `
        -Department $Department `
        -Company $Company `
        -Manager $Manager `
        -Path $TargetOU `
        -AccountPassword $TempPassword `
        -ChangePasswordAtLogon $true `
        -Enabled $true `
        -ErrorAction Stop

    Write-Host "OK: User account '$Username' created successfully." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Failed to create user account." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 4: Add to HR-Users security group
Write-Host "Step 4 - Adding to '$SecurityGroup' security group..." -ForegroundColor Yellow

try {
    Add-ADGroupMember -Identity $SecurityGroup -Members $Username -ErrorAction Stop
    Write-Host "OK: '$Username' added to '$SecurityGroup'." -ForegroundColor Green
} catch {
    Write-Host "WARNING: Could not add user to '$SecurityGroup'." -ForegroundColor Yellow
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}

Write-Host ""

# Step 5: Verify and summarise
Write-Host "Step 5 - Verifying account..." -ForegroundColor Yellow

$NewUser = Get-ADUser -Identity $Username -Properties DisplayName, Title, Department, DistinguishedName, Enabled

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Account Created Successfully" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Display Name : $($NewUser.DisplayName)"
Write-Host "Username     : $($NewUser.SamAccountName)"
Write-Host "Title        : $($NewUser.Title)"
Write-Host "Department   : $($NewUser.Department)"
Write-Host "Enabled      : $($NewUser.Enabled)"
Write-Host "OU Path      : $($NewUser.DistinguishedName)"
Write-Host ""
Write-Host "Temp Password   : TempPassword123!" -ForegroundColor Yellow
Write-Host "Password Change : Required at first logon" -ForegroundColor Yellow
Write-Host ""
Write-Host "Next step: Have the user log into WS-01 to verify access." -ForegroundColor Cyan
Write-Host ""
```

---

## New-ITUser.ps1

Creates an IT department user. Places the user in `IT\Users`, assigns to both `IT-Users` and `Helpdesk-Admins`, sets temp password with forced change at first logon. The dual group assignment grants delegated AD control across all department OUs without Domain Admin rights.

**Usage:**
```powershell
.\New-ITUser.ps1 -FirstName "Michael" -LastName "Cabrera" -JobTitle "IT Support Specialist"
```

```powershell
# New-ITUser.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates a new Active Directory user account locked to the IT department.
# Assigns to IT-Users and Helpdesk-Admins security groups, sets temp password,
# forces password change at first logon, and populates all standard AD attributes.
# Run on DC-01 with Domain Admin privileges.

param (
    [Parameter(Mandatory = $true)]
    [string]$FirstName,

    [Parameter(Mandatory = $true)]
    [string]$LastName,

    [Parameter(Mandatory = $true)]
    [string]$JobTitle
)

# --- Configuration ---
$Department    = "IT"
$Manager       = "paul.atreides"
$TargetOU      = "OU=Users,OU=IT,OU=Departments,DC=lab,DC=local"
$SecurityGroup = "IT-Users"
$HelpdeskGroup = "Helpdesk-Admins"
$TempPassword  = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force
$Company       = "lab.local"

# --- Derived Values ---
$Username    = "$($FirstName.ToLower()).$($LastName.ToLower())"
$DisplayName = "$FirstName $LastName"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES - IT User Onboarding" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Name       : $DisplayName"
Write-Host "Username   : $Username"
Write-Host "Title      : $JobTitle"
Write-Host "Department : $Department"
Write-Host "OU         : $TargetOU"
Write-Host "Groups     : $SecurityGroup, $HelpdeskGroup"
Write-Host ""

# Step 1: Check if user already exists
Write-Host "Step 1 - Checking if user already exists..." -ForegroundColor Yellow

try {
    Get-ADUser -Identity $Username -ErrorAction Stop | Out-Null
    Write-Host "ERROR: User '$Username' already exists in Active Directory." -ForegroundColor Red
    exit 1
} catch {
    Write-Host "OK: Username '$Username' is available." -ForegroundColor Green
}

Write-Host ""

# Step 2: Verify target OU exists
Write-Host "Step 2 - Verifying target OU exists..." -ForegroundColor Yellow

try {
    Get-ADOrganizationalUnit -Identity $TargetOU -ErrorAction Stop | Out-Null
    Write-Host "OK: Target OU found." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Target OU not found: $TargetOU" -ForegroundColor Red
    Write-Host "Run New-OUStructure.ps1 first to create the OU structure." -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 3: Create the user account
Write-Host "Step 3 - Creating user account..." -ForegroundColor Yellow

try {
    New-ADUser `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@lab.local" `
        -GivenName $FirstName `
        -Surname $LastName `
        -DisplayName $DisplayName `
        -Name $DisplayName `
        -Title $JobTitle `
        -Department $Department `
        -Company $Company `
        -Manager $Manager `
        -Path $TargetOU `
        -AccountPassword $TempPassword `
        -ChangePasswordAtLogon $true `
        -Enabled $true `
        -ErrorAction Stop

    Write-Host "OK: User account '$Username' created successfully." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Failed to create user account." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 4: Add to IT-Users security group
Write-Host "Step 4 - Adding to '$SecurityGroup' security group..." -ForegroundColor Yellow

try {
    Add-ADGroupMember -Identity $SecurityGroup -Members $Username -ErrorAction Stop
    Write-Host "OK: '$Username' added to '$SecurityGroup'." -ForegroundColor Green
} catch {
    Write-Host "WARNING: Could not add user to '$SecurityGroup'." -ForegroundColor Yellow
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}

Write-Host ""

# Step 5: Add to Helpdesk-Admins group
Write-Host "Step 5 - Adding to '$HelpdeskGroup' group..." -ForegroundColor Yellow

try {
    Add-ADGroupMember -Identity $HelpdeskGroup -Members $Username -ErrorAction Stop
    Write-Host "OK: '$Username' added to '$HelpdeskGroup'." -ForegroundColor Green
    Write-Host "Note: Helpdesk-Admins grants delegated AD control across all department OUs." -ForegroundColor Cyan
} catch {
    Write-Host "WARNING: Could not add user to '$HelpdeskGroup'." -ForegroundColor Yellow
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}

Write-Host ""

# Step 6: Verify and summarise
Write-Host "Step 6 - Verifying account..." -ForegroundColor Yellow

$NewUser = Get-ADUser -Identity $Username -Properties DisplayName, Title, Department, DistinguishedName, Enabled
$Groups  = Get-ADPrincipalGroupMembership -Identity $Username | Select-Object -ExpandProperty Name

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Account Created Successfully" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Display Name : $($NewUser.DisplayName)"
Write-Host "Username     : $($NewUser.SamAccountName)"
Write-Host "Title        : $($NewUser.Title)"
Write-Host "Department   : $($NewUser.Department)"
Write-Host "Enabled      : $($NewUser.Enabled)"
Write-Host "OU Path      : $($NewUser.DistinguishedName)"
Write-Host "Groups       : $($Groups -join ', ')"
Write-Host ""
Write-Host "Temp Password   : TempPassword123!" -ForegroundColor Yellow
Write-Host "Password Change : Required at first logon" -ForegroundColor Yellow
Write-Host ""
Write-Host "Next step: Have the user log into WS-01 to verify access." -ForegroundColor Cyan
Write-Host ""
```

---

## New-SalesUser.ps1

Creates a Sales department user. Places the user in `Sales\Users`, assigns to `Sales-Users`, sets temp password with forced change at first logon.

**Usage:**
```powershell
.\New-SalesUser.ps1 -FirstName "Khoran" -LastName "Gomez" -JobTitle "Sales Associate"
```

```powershell
# New-SalesUser.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates a new Active Directory user account locked to the Sales department.
# Assigns to Sales-Users security group, sets temp password, forces password
# change at first logon, and populates all standard AD attributes.
# Run on DC-01 with Domain Admin privileges.

param (
    [Parameter(Mandatory = $true)]
    [string]$FirstName,

    [Parameter(Mandatory = $true)]
    [string]$LastName,

    [Parameter(Mandatory = $true)]
    [string]$JobTitle
)

# --- Configuration ---
$Department    = "Sales"
$Manager       = "anakin.skywalker"
$TargetOU      = "OU=Users,OU=Sales,OU=Departments,DC=lab,DC=local"
$SecurityGroup = "Sales-Users"
$TempPassword  = ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force
$Company       = "lab.local"

# --- Derived Values ---
$Username    = "$($FirstName.ToLower()).$($LastName.ToLower())"
$DisplayName = "$FirstName $LastName"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES - Sales User Onboarding" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Name       : $DisplayName"
Write-Host "Username   : $Username"
Write-Host "Title      : $JobTitle"
Write-Host "Department : $Department"
Write-Host "OU         : $TargetOU"
Write-Host "Group      : $SecurityGroup"
Write-Host ""

# Step 1: Check if user already exists
Write-Host "Step 1 - Checking if user already exists..." -ForegroundColor Yellow

try {
    Get-ADUser -Identity $Username -ErrorAction Stop | Out-Null
    Write-Host "ERROR: User '$Username' already exists in Active Directory." -ForegroundColor Red
    exit 1
} catch {
    Write-Host "OK: Username '$Username' is available." -ForegroundColor Green
}

Write-Host ""

# Step 2: Verify target OU exists
Write-Host "Step 2 - Verifying target OU exists..." -ForegroundColor Yellow

try {
    Get-ADOrganizationalUnit -Identity $TargetOU -ErrorAction Stop | Out-Null
    Write-Host "OK: Target OU found." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Target OU not found: $TargetOU" -ForegroundColor Red
    Write-Host "Run New-OUStructure.ps1 first to create the OU structure." -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 3: Create the user account
Write-Host "Step 3 - Creating user account..." -ForegroundColor Yellow

try {
    New-ADUser `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@lab.local" `
        -GivenName $FirstName `
        -Surname $LastName `
        -DisplayName $DisplayName `
        -Name $DisplayName `
        -Title $JobTitle `
        -Department $Department `
        -Company $Company `
        -Manager $Manager `
        -Path $TargetOU `
        -AccountPassword $TempPassword `
        -ChangePasswordAtLogon $true `
        -Enabled $true `
        -ErrorAction Stop

    Write-Host "OK: User account '$Username' created successfully." -ForegroundColor Green
} catch {
    Write-Host "ERROR: Failed to create user account." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    exit 1
}

Write-Host ""

# Step 4: Add to Sales-Users security group
Write-Host "Step 4 - Adding to '$SecurityGroup' security group..." -ForegroundColor Yellow

try {
    Add-ADGroupMember -Identity $SecurityGroup -Members $Username -ErrorAction Stop
    Write-Host "OK: '$Username' added to '$SecurityGroup'." -ForegroundColor Green
} catch {
    Write-Host "WARNING: Could not add user to '$SecurityGroup'." -ForegroundColor Yellow
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}

Write-Host ""

# Step 5: Verify and summarise
Write-Host "Step 5 - Verifying account..." -ForegroundColor Yellow

$NewUser = Get-ADUser -Identity $Username -Properties DisplayName, Title, Department, DistinguishedName, Enabled

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Account Created Successfully" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Display Name : $($NewUser.DisplayName)"
Write-Host "Username     : $($NewUser.SamAccountName)"
Write-Host "Title        : $($NewUser.Title)"
Write-Host "Department   : $($NewUser.Department)"
Write-Host "Enabled      : $($NewUser.Enabled)"
Write-Host "OU Path      : $($NewUser.DistinguishedName)"
Write-Host ""
Write-Host "Temp Password   : TempPassword123!" -ForegroundColor Yellow
Write-Host "Password Change : Required at first logon" -ForegroundColor Yellow
Write-Host ""
Write-Host "Next step: Have the user log into WS-01 to verify access." -ForegroundColor Cyan
Write-Host ""
```

---

## New-DeptShares.ps1

Creates the four department SMB shares on DC-01. For each department: creates the folder at `C:\Shares\[Dept]`, removes inherited permissions, sets explicit NTFS permissions, creates the SMB share, and applies share-level permissions. Removes `Everyone` from all shares.

**Usage:**
```powershell
.\New-DeptShares.ps1
```

```powershell
# New-DeptShares.ps1
# HERMES — Helpdesk Ticketing and Automation
# Creates all department shared folders on DC-01 at C:\Shares\
# Sets share-level and NTFS permissions tied to AD security groups.
# Removes Everyone and Authenticated Users from all shares.
# Helpdesk-Admins receive Read access on all shares for troubleshooting.
# Run on DC-01 with Domain Admin privileges.

# --- Configuration ---
$ShareRoot = "C:\Shares"

$Departments = @(
    @{ Name = "Finance"; Group = "Finance-Users" },
    @{ Name = "HR";      Group = "HR-Users"      },
    @{ Name = "IT";      Group = "IT-Users"      },
    @{ Name = "Sales";   Group = "Sales-Users"   }
)

$HelpdeskGroup = "Helpdesk-Admins"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  HERMES - Department Share Setup" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

# Step 1: Create root Shares folder if it doesn't exist
Write-Host "Step 1 - Checking root share folder..." -ForegroundColor Yellow

if (-not (Test-Path $ShareRoot)) {
    New-Item -Path $ShareRoot -ItemType Directory -Force | Out-Null
    Write-Host "OK: Created $ShareRoot" -ForegroundColor Green
} else {
    Write-Host "OK: $ShareRoot already exists." -ForegroundColor Green
}

Write-Host ""

# Step 2: Create each department folder, share, and set permissions
$Created = 0
$Skipped = 0
$Failed  = 0

foreach ($Dept in $Departments) {

    $FolderPath = "$ShareRoot\$($Dept.Name)"
    $ShareName  = $Dept.Name
    $DeptGroup  = $Dept.Group

    Write-Host "Processing: $($Dept.Name)" -ForegroundColor Yellow

    # Create folder
    if (-not (Test-Path $FolderPath)) {
        try {
            New-Item -Path $FolderPath -ItemType Directory -Force | Out-Null
            Write-Host "  OK: Folder created — $FolderPath" -ForegroundColor Green
        } catch {
            Write-Host "  ERROR: Failed to create folder — $FolderPath" -ForegroundColor Red
            $Failed++
            Write-Host ""
            continue
        }
    } else {
        Write-Host "  OK: Folder already exists — $FolderPath" -ForegroundColor Green
    }

    # Set NTFS permissions
    try {
        $Acl = Get-Acl -Path $FolderPath
        $Acl.SetAccessRuleProtection($true, $false)
        $Acl.Access | ForEach-Object { $Acl.RemoveAccessRule($_) | Out-Null }

        $Acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
            "SYSTEM", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))

        $Acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
            "Domain Admins", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))

        $Acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
            $DeptGroup, "Modify", "ContainerInherit,ObjectInherit", "None", "Allow")))

        $Acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
            $HelpdeskGroup, "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")))

        Set-Acl -Path $FolderPath -AclObject $Acl
        Write-Host "  OK: NTFS permissions set — $DeptGroup (Modify), $HelpdeskGroup (Read)" -ForegroundColor Green
    } catch {
        Write-Host "  ERROR: Failed to set NTFS permissions on $FolderPath" -ForegroundColor Red
        $Failed++
        Write-Host ""
        continue
    }

    # Create SMB share
    try {
        $ExistingShare = Get-SmbShare -Name $ShareName -ErrorAction SilentlyContinue

        if ($ExistingShare) {
            Write-Host "  OK: Share '$ShareName' already exists — skipping." -ForegroundColor Yellow
            $Skipped++
        } else {
            New-SmbShare `
                -Name $ShareName `
                -Path $FolderPath `
                -FullAccess "Domain Admins" `
                -ChangeAccess $DeptGroup `
                -ReadAccess $HelpdeskGroup `
                -ErrorAction Stop | Out-Null

            Revoke-SmbShareAccess -Name $ShareName -AccountName "Everyone" -Force -ErrorAction SilentlyContinue | Out-Null

            Write-Host "  OK: Share '\\DC-01\$ShareName' created." -ForegroundColor Green
            Write-Host "  Share permissions: $DeptGroup (Change), $HelpdeskGroup (Read), Domain Admins (Full)" -ForegroundColor Cyan
            $Created++
        }
    } catch {
        Write-Host "  ERROR: Failed to create share '$ShareName'." -ForegroundColor Red
        $Failed++
    }

    Write-Host ""
}

# Summary
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Department Share Setup Complete" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "  Shares Created : $Created" -ForegroundColor Green
Write-Host "  Shares Skipped : $Skipped" -ForegroundColor Yellow
Write-Host "  Shares Failed  : $Failed" -ForegroundColor $(if ($Failed -gt 0) { "Red" } else { "Green" })
Write-Host ""
Write-Host "Share paths:" -ForegroundColor Cyan
Write-Host "  \\DC-01\Finance  — Finance-Users (Modify)"
Write-Host "  \\DC-01\HR       — HR-Users (Modify)"
Write-Host "  \\DC-01\IT       — IT-Users (Modify)"
Write-Host "  \\DC-01\Sales    — Sales-Users (Modify)"
Write-Host ""
Write-Host "Next steps:" -ForegroundColor Cyan
Write-Host "  - Verify shares are accessible from WS-01"
Write-Host "  - Confirm cross-department access is denied"
Write-Host "  - Close the ticket once user confirms access"
Write-Host ""
```
