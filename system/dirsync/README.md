# Active Directory DirSync Permissions Management Guide

This guide provides step-by-step instructions for managing Directory Synchronization (DirSync) permissions in Active Directory. DirSync permissions allow accounts to replicate directory changes, which is essential for Azure AD Connect and similar synchronization tools.[1][11]

## Overview

The two critical permissions required for directory synchronization are:
- **DS-Replication-Get-Changes** (GUID: `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`)
- **DS-Replication-Get-Changes-All** (GUID: `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`)

These permissions are granted at the domain root level and allow an account to read directory changes for synchronization purposes.[12][1]

---

## Table of Contents

1. [List Current DirSync Users and Computers](#1-list-current-dirsync-users-and-computers)
2. [Add DirSync Permissions for User](#2-add-dirsync-permissions-for-user)
3. [Add DirSync Permissions for Computer](#3-add-dirsync-permissions-for-computer)
4. [Remove DirSync Permissions from User](#4-remove-dirsync-permissions-from-user)
5. [Remove DirSync Permissions from Computer](#5-remove-dirsync-permissions-from-computer)
6. [Configure Read-Only Permissions](#6-configure-read-only-permissions)
7. [List Read-Only Permissions](#7-list-read-only-permissions)
8. [Add Read-Only Permissions](#8-add-read-only-permissions)
9. [Remove Read-Only Permissions](#9-remove-read-only-permissions)

***

## 1. List Current DirSync Users and Computers

Use this script to view all accounts (users and computers) that currently have DirSync permissions on your domain.

### Script: `List-DirSync.ps1`

```powershell
<#
.SYNOPSIS
    Lists all users and computers with DirSync permissions.

.DESCRIPTION
    Queries the domain root ACL and identifies accounts with
    DS-Replication-Get-Changes or DS-Replication-Get-Changes-All permissions.
#>

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

Write-Host "`n=== DirSync Permissions Report ===" -ForegroundColor Cyan
Write-Host "Domain: $domainDN`n" -ForegroundColor Cyan

$foundPermissions = $false

# Find all ACEs with DirSync permissions
$dirSyncACEs = $acl.Access | Where-Object {
    ($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)
} | Select-Object IdentityReference, ObjectType, AccessControlType | Sort-Object IdentityReference -Unique

foreach ($ace in $dirSyncACEs) {
    $foundPermissions = $true
    $identity = $ace.IdentityReference
    $permType = if ($ace.ObjectType -eq $guidRepChanges) { "DS-Replication-Get-Changes" } else { "DS-Replication-Get-Changes-All" }
    
    Write-Host "Identity: $identity" -ForegroundColor Green
    Write-Host "  Permission: $permType" -ForegroundColor Yellow
    Write-Host "  Access: $($ace.AccessControlType)`n"
}

if (-not $foundPermissions) {
    Write-Host "No DirSync permissions found on this domain." -ForegroundColor Yellow
}
```

### Usage:
```powershell
.\List-DirSync.ps1
```

**Expected Output:**
```
=== DirSync Permissions Report ===
Domain: DC=contoso,DC=com

Identity: CONTOSO\adel.ali
  Permission: DS-Replication-Get-Changes
  Access: Allow

Identity: CONTOSO\AALY-LSMH$
  Permission: DS-Replication-Get-Changes-All
  Access: Allow
```

***

## 2. Add DirSync Permissions for User

This script grants DirSync permissions to a user account. In this example, we use **adel.ali**.[1][12]

### Script: `Add-DirSync-User.ps1`

```powershell
<#
.SYNOPSIS
    Grants DirSync permissions to user: adel.ali

.DESCRIPTION
    Adds DS-Replication-Get-Changes and DS-Replication-Get-Changes-All
    permissions to the specified user account at the domain root level.
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

# Get user object and SID
$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# DirSync permission GUIDs
$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

# Create and add DS-Replication-Get-Changes permission
$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChanges
)
$acl.AddAccessRule($ace1)

# Create and add DS-Replication-Get-Changes-All permission
$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChangesAll
)
$acl.AddAccessRule($ace2)

# Apply changes
Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ DirSync permissions granted to user: $Identity" -ForegroundColor Green
Write-Host "   - DS-Replication-Get-Changes" -ForegroundColor Yellow
Write-Host "   - DS-Replication-Get-Changes-All" -ForegroundColor Yellow
```

### Usage:
```powershell
.\Add-DirSync-User.ps1
```

**Result:** User **adel.ali** now has DirSync permissions on the domain.[12][1]

***

## 3. Add DirSync Permissions for Computer

This script grants DirSync permissions to a computer account. In this example, we use **AALY-LSMH**.[1]

### Script: `Add-DirSync-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Grants DirSync permissions to computer: AALY-LSMH

.DESCRIPTION
    Adds DS-Replication-Get-Changes and DS-Replication-Get-Changes-All
    permissions to the specified computer account at the domain root level.
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

# Get computer object and SID
$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# DirSync permission GUIDs
$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

# Create and add DS-Replication-Get-Changes permission
$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChanges
)
$acl.AddAccessRule($ace1)

# Create and add DS-Replication-Get-Changes-All permission
$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChangesAll
)
$acl.AddAccessRule($ace2)

# Apply changes
Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ DirSync permissions granted to computer: $Identity" -ForegroundColor Green
Write-Host "   - DS-Replication-Get-Changes" -ForegroundColor Yellow
Write-Host "   - DS-Replication-Get-Changes-All" -ForegroundColor Yellow
```

### Usage:
```powershell
.\Add-DirSync-Computer.ps1
```

**Result:** Computer **AALY-LSMH** now has DirSync permissions on the domain.[1]

***

## 4. Remove DirSync Permissions from User

This script removes DirSync permissions from a user account. In this example, we remove permissions from **adel.ali**.[12][1]

### Script: `Remove-DirSync-User.ps1`

```powershell
<#
.SYNOPSIS
    Removes DirSync permissions from user: adel.ali

.DESCRIPTION
    Removes DS-Replication-Get-Changes and DS-Replication-Get-Changes-All
    permissions from the specified user account at the domain root level.
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

# Get user object and SID
$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# DirSync permission GUIDs
$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

function Remove-IfExists {
    param($acl, $sid, $guid)
    $matches = $acl.Access | Where-Object {
        ($_.ObjectType -eq $guid) -and ($_.IdentityReference -eq $sid)
    }
    $removed = $false
    foreach ($m in $matches) {
        # Create identical ACE object to remove
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $sid,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guid
        )
        if ($acl.RemoveAccessRule($ace)) { $removed = $true }
    }
    return $removed
}

$changed = $false
if (Remove-IfExists -acl $acl -sid $sid -guid $guidRepChanges) { $changed = $true }
if (Remove-IfExists -acl $acl -sid $sid -guid $guidRepChangesAll) { $changed = $true }

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "✅ DirSync permissions removed from user: $Identity" -ForegroundColor Green
} else {
    Write-Host "ℹ️  No DirSync permissions found for user: $Identity" -ForegroundColor Cyan
}
```

### Usage:
```powershell
.\Remove-DirSync-User.ps1
```

**Result:** User **adel.ali** no longer has DirSync permissions on the domain.[12][1]

***

## 5. Remove DirSync Permissions from Computer

This script removes DirSync permissions from a computer account. In this example, we remove permissions from **AALY-LSMH**.[1]

### Script: `Remove-DirSync-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Removes DirSync permissions from computer: AALY-LSMH

.DESCRIPTION
    Removes DS-Replication-Get-Changes and DS-Replication-Get-Changes-All
    permissions from the specified computer account at the domain root level.
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

# Get computer object and SID
$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# DirSync permission GUIDs
$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

function Remove-IfExists {
    param($acl, $sid, $guid)
    $matches = $acl.Access | Where-Object {
        ($_.ObjectType -eq $guid) -and ($_.IdentityReference -eq $sid)
    }
    $removed = $false
    foreach ($m in $matches) {
        # Create identical ACE object to remove
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $sid,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guid
        )
        if ($acl.RemoveAccessRule($ace)) { $removed = $true }
    }
    return $removed
}

$changed = $false
if (Remove-IfExists -acl $acl -sid $sid -guid $guidRepChanges) { $changed = $true }
if (Remove-IfExists -acl $acl -sid $sid -guid $guidRepChangesAll) { $changed = $true }

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "✅ DirSync permissions removed from computer: $Identity" -ForegroundColor Green
} else {
    Write-Host "ℹ️  No DirSync permissions found for computer: $Identity" -ForegroundColor Cyan
}
```

### Usage:
```powershell
.\Remove-DirSync-Computer.ps1
```

**Result:** Computer **AALY-LSMH** no longer has DirSync permissions on the domain.[1]

***

## 6. Configure Read-Only Permissions

Read-only permissions allow an account to read all directory properties without the ability to replicate sensitive data like password hashes. This is useful for monitoring, reporting, or basic synchronization scenarios where you don't need full DirSync permissions.[2][1]

### Understanding Read-Only Permissions

Basic read-only permissions grant the following access:[1]

| Type | Access | Applies To |
|------|--------|------------|
| Allow | Read all properties | Descendant device objects |
| Allow | Read all properties | Descendant InetOrgPerson objects |
| Allow | Read all properties | Descendant Computer objects |
| Allow | Read all properties | Descendant foreignSecurityPrincipal objects |
| Allow | Read all properties | Descendant Group objects |
| Allow | Read all properties | Descendant User objects |
| Allow | Read all properties | Descendant Contact objects |

These permissions are safer than full DirSync permissions because they don't allow password hash synchronization or replication of all directory changes.[2][1]

***

## 7. List Read-Only Permissions

This script lists all accounts with read-only permissions on domain objects.

### Script: `List-ReadOnly-Permissions.ps1`

```powershell
<#
.SYNOPSIS
    Lists all users and computers with read-only permissions on AD objects.

.DESCRIPTION
    Queries the domain root ACL and identifies accounts with
    GenericRead or ReadProperty permissions on descendant objects.
#>

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:$domainDN"

Write-Host "`n=== Read-Only Permissions Report ===" -ForegroundColor Cyan
Write-Host "Domain: $domainDN`n" -ForegroundColor Cyan

$foundPermissions = $false

# Find all ACEs with Read permissions
$readOnlyACEs = $acl.Access | Where-Object {
    ($_.ActiveDirectoryRights -match "ReadProperty|GenericRead") -and
    ($_.AccessControlType -eq "Allow") -and
    ($_.IdentityReference -notlike "*BUILTIN*") -and
    ($_.IdentityReference -notlike "*NT AUTHORITY*") -and
    ($_.IdentityReference -notlike "*S-1-5-*")
} | Select-Object IdentityReference, ActiveDirectoryRights, InheritanceType -Unique

foreach ($ace in $readOnlyACEs) {
    $foundPermissions = $true
    $identity = $ace.IdentityReference
    
    Write-Host "Identity: $identity" -ForegroundColor Green
    Write-Host "  Rights: $($ace.ActiveDirectoryRights)" -ForegroundColor Yellow
    Write-Host "  Inheritance: $($ace.InheritanceType)`n"
}

if (-not $foundPermissions) {
    Write-Host "No custom read-only permissions found on this domain." -ForegroundColor Yellow
}
```

### Usage:
```powershell
.\List-ReadOnly-Permissions.ps1
```

**Expected Output:**
```
=== Read-Only Permissions Report ===
Domain: DC=contoso,DC=com

Identity: CONTOSO\adel.ali
  Rights: ReadProperty, GenericRead
  Inheritance: Descendents
```

***

## 8. Add Read-Only Permissions

This script grants basic read-only permissions to a user account. In this example, we use **adel.ali**.[2][1]

### Script: `Add-ReadOnly-User.ps1`

```powershell
<#
.SYNOPSIS
    Grants read-only permissions to user: adel.ali

.DESCRIPTION
    Adds read all properties permission to the specified user account
    for all descendant objects (users, computers, groups, contacts, devices).
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

# Get user object and SID
$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# Define object type GUIDs for different AD object classes
$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "Device"                   = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

# Add read permissions for each object type
foreach ($objectType in $objectTypes.Keys) {
    $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
        $sid,
        [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty,
        [System.Security.AccessControl.AccessControlType]::Allow,
        [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
        $objectTypes[$objectType]
    )
    $acl.AddAccessRule($ace)
}

# Apply changes
Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ Read-only permissions granted to user: $Identity" -ForegroundColor Green
Write-Host "   - Read all properties on User objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on Computer objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on Group objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on Contact objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on Device objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on InetOrgPerson objects" -ForegroundColor Yellow
Write-Host "   - Read all properties on ForeignSecurityPrincipal objects" -ForegroundColor Yellow
```

### Usage:
```powershell
.\Add-ReadOnly-User.ps1
```

**Result:** User **adel.ali** now has read-only permissions on all AD object types.[2][1]

***

## 9. Remove Read-Only Permissions

This script removes read-only permissions from a user account. In this example, we remove permissions from **adel.ali**.[1]

### Script: `Remove-ReadOnly-User.ps1`

```powershell
<#
.SYNOPSIS
    Removes read-only permissions from user: adel.ali

.DESCRIPTION
    Removes read all properties permissions from the specified user account
    for all descendant objects.
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

# Get user object and SID
$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID

# Get domain root ACL
$acl = Get-Acl "AD:$domainDN"

# Define object type GUIDs for different AD object classes
$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "Device"                   = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

$changed = $false

# Remove read permissions for each object type
foreach ($objectType in $objectTypes.Keys) {
    $matches = $acl.Access | Where-Object {
        ($_.IdentityReference -eq $sid) -and
        ($_.InheritedObjectType -eq $objectTypes[$objectType]) -and
        ($_.ActiveDirectoryRights -match "ReadProperty")
    }
    
    foreach ($match in $matches) {
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $sid,
            [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty,
            [System.Security.AccessControl.AccessControlType]::Allow,
            [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
            $objectTypes[$objectType]
        )
        if ($acl.RemoveAccessRule($ace)) {
            $changed = $true
        }
    }
}

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "✅ Read-only permissions removed from user: $Identity" -ForegroundColor Green
} else {
    Write-Host "ℹ️  No read-only permissions found for user: $Identity" -ForegroundColor Cyan
}
```

### Usage:
```powershell
.\Remove-ReadOnly-User.ps1
```

**Result:** User **adel.ali** no longer has read-only permissions on the domain.[1]

---

## Prerequisites

- **PowerShell ActiveDirectory Module** installed
- **Domain Administrator** or **Enterprise Administrator** privileges
- PowerShell execution policy allowing script execution

To check if the ActiveDirectory module is available:
```powershell
Import-Module ActiveDirectory
```

To install the module if missing:
```powershell
Install-WindowsFeature RSAT-AD-PowerShell
```

***

## Permission Comparison

| Permission Type | Purpose | Security Level | Use Case |
|----------------|---------|----------------|----------|
| **Read-Only** | Basic directory read access | Low risk | Monitoring, reporting, basic inventory [1][2] |
| **DirSync (Get-Changes)** | Read directory changes | Medium risk | Basic synchronization without passwords [1] |
| **DirSync (Get-Changes-All)** | Read all changes including passwords | High risk | Full Azure AD Connect sync [1][12] |

***

## Security Considerations

DirSync permissions are powerful and should only be granted to trusted service accounts used for directory synchronization (such as Azure AD Connect service accounts). These permissions allow reading password hashes and all directory change information, making them a potential security risk if misused.[11][12][1]

### Best Practices

**For DirSync Permissions:**
- Only grant DirSync permissions to dedicated service accounts
- Use read-only permissions when password synchronization is not required[2][1]
- Regularly audit accounts with these permissions using the List scripts
- Use strong passwords and enable MFA for accounts with these permissions
- Monitor for unauthorized privilege escalation attempts[11]

**For Read-Only Permissions:**
- Grant read-only access for monitoring and reporting scenarios[2]
- Read-only permissions do not allow password hash access, making them safer[1]
- Combine read-only domain access with write access to specific OUs when needed[2]
- Consider using read-only permissions for integration and inventory tools[2]

---

## Troubleshooting

### Access Denied Errors

If you encounter "Access Denied" errors, ensure:
1. You're running PowerShell as Administrator
2. Your account has Domain Admin or Enterprise Admin rights
3. The ActiveDirectory PowerShell module is properly installed

### Account Not Found

If the scripts don't find the account, verify:
1. The account name is spelled correctly (use `Get-ADUser -Identity "username"` to test)
2. The account exists in Active Directory
3. For computer accounts, ensure the computer object exists in AD[12][1]

### Permissions Not Taking Effect

If permissions don't seem to work:
1. Wait a few minutes for AD replication
2. Restart the service using the account (e.g., Azure AD Connect)
3. Verify permissions using the List scripts
4. Check Event Viewer for related errors[1]

***

## Additional Resources

- [Microsoft Entra Connect: Configure AD DS Connector Account Permissions](https://learn.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-configure-ad-ds-connector-account)[1]
- [Active Directory Synchronization Best Practices](https://docs.microsoft.com)[2]

***

**Document Version:** 1.0  
**Last Updated:** November 11, 2025  
**Tested On:** Windows Server 2016/2019/2022, Active Directory Domain Services
