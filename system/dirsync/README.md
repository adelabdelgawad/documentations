# Active Directory DirSync & Read-Only Permissions Management Guide

Complete guide for managing Directory Synchronization (DirSync) and Read-Only permissions in Active Directory for both users and computers.[1][2][3]

## Overview

This guide provides comprehensive PowerShell scripts to manage two types of permissions:

### DirSync Permissions
- **DS-Replication-Get-Changes** (GUID: `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`)
- **DS-Replication-Get-Changes-All** (GUID: `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`)

### Read-Only Permissions
- **ReadProperty** on User, Computer, Group, Contact, InetOrgPerson, and ForeignSecurityPrincipal objects

***

## Table of Contents

1. [Universal Management Scripts](#1-universal-management-scripts)
2. [List All Permissions](#2-list-all-permissions)
3. [Add DirSync Permissions](#3-add-dirsync-permissions)
4. [Add Read-Only Permissions](#4-add-read-only-permissions)
5. [Remove DirSync Permissions](#5-remove-dirsync-permissions)
6. [Remove Read-Only Permissions](#6-remove-read-only-permissions)
7. [Check Specific Account Permissions](#7-check-specific-account-permissions)

***

## 1. Universal Management Scripts

### Master Permission Manager

This comprehensive script manages both DirSync and Read-Only permissions for users and computers.[2][1]

### Script: `Manage-ADPermissions.ps1`

```powershell
<#
.SYNOPSIS
    Universal Active Directory Permissions Manager

.DESCRIPTION
    Manages DirSync and Read-Only permissions for users and computers.
    
.PARAMETER Identity
    The username or computer name (without $)
    
.PARAMETER Type
    User or Computer
    
.PARAMETER Action
    Add-DirSync, Add-ReadOnly, Remove-DirSync, Remove-ReadOnly, Check, List
    
.EXAMPLE
    .\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync
    .\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-ReadOnly
    .\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check
    .\Manage-ADPermissions.ps1 -Action List
#>

param(
    [Parameter(Mandatory=$false)]
    [string]$Identity,
    
    [Parameter(Mandatory=$false)]
    [ValidateSet("User","Computer")]
    [string]$Type,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet("Add-DirSync","Add-ReadOnly","Remove-DirSync","Remove-ReadOnly","Check","List")]
    [string]$Action
)

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

# DirSync permission GUIDs
$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

# Object type GUIDs
$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

$objectTypeNames = @{
    "bf967aba-0de6-11d0-a285-00aa003049e2" = "User"
    "bf967a86-0de6-11d0-a285-00aa003049e2" = "Computer"
    "bf967a9c-0de6-11d0-a285-00aa003049e2" = "Group"
    "5cb41ed0-0e4c-11d0-a286-00aa003049e2" = "Contact"
    "4828cc14-1437-45bc-9b07-ad6f015e5f28" = "InetOrgPerson"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "ForeignSecurityPrincipal"
}

function Get-AccountInfo {
    param($Identity, $Type)
    
    try {
        if ($Type -eq "User") {
            $obj = Get-ADUser -Identity $Identity -ErrorAction Stop
        } else {
            $obj = Get-ADComputer -Identity $Identity -ErrorAction Stop
        }
        
        $sid = $obj.SID
        $ntAccount = $sid.Translate([System.Security.Principal.NTAccount])
        
        return @{
            Object = $obj
            SID = $sid
            NTAccount = $ntAccount
            Success = $true
        }
    }
    catch {
        Write-Host "❌ Error: Could not find $Type '$Identity'" -ForegroundColor Red
        Write-Host "   $($_.Exception.Message)" -ForegroundColor Red
        return @{ Success = $false }
    }
}

function Add-DirSyncPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    $acl = Get-Acl "AD:$domainDN"
    
    # Add DS-Replication-Get-Changes
    $ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
        $accountInfo.SID,
        [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
        [System.Security.AccessControl.AccessControlType]::Allow,
        $guidRepChanges
    )
    $acl.AddAccessRule($ace1)
    
    # Add DS-Replication-Get-Changes-All
    $ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
        $accountInfo.SID,
        [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
        [System.Security.AccessControl.AccessControlType]::Allow,
        $guidRepChangesAll
    )
    $acl.AddAccessRule($ace2)
    
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    
    Write-Host "✅ DirSync permissions granted to $Type`: $Identity" -ForegroundColor Green
    Write-Host "   - DS-Replication-Get-Changes" -ForegroundColor Yellow
    Write-Host "   - DS-Replication-Get-Changes-All" -ForegroundColor Yellow
}

function Add-ReadOnlyPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    $acl = Get-Acl "AD:$domainDN"
    
    foreach ($objectType in $objectTypes.Keys) {
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $accountInfo.SID,
            [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty,
            [System.Security.AccessControl.AccessControlType]::Allow,
            [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
            $objectTypes[$objectType]
        )
        $acl.AddAccessRule($ace)
    }
    
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    
    Write-Host "✅ Read-Only permissions granted to $Type`: $Identity" -ForegroundColor Green
    foreach ($objectType in $objectTypes.Keys) {
        Write-Host "   - ReadProperty on $objectType objects" -ForegroundColor Yellow
    }
}

function Remove-DirSyncPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    $acl = Get-Acl "AD:$domainDN"
    $changed = $false
    
    $acesToRemove = $acl.Access | Where-Object {
        (($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)) -and
        (($_.IdentityReference.Value -eq $accountInfo.SID.Value) -or 
         ($_.IdentityReference.Value -eq $accountInfo.NTAccount.Value))
    }
    
    foreach ($ace in $acesToRemove) {
        $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
            "DS-Replication-Get-Changes" 
        } else { 
            "DS-Replication-Get-Changes-All" 
        }
        Write-Host "Removing: $($ace.IdentityReference) - $permType" -ForegroundColor Yellow
        $acl.RemoveAccessRuleSpecific($ace)
        $changed = $true
    }
    
    if ($changed) {
        Set-Acl -Path "AD:$domainDN" -AclObject $acl
        Write-Host "`n✅ DirSync permissions removed from $Type`: $Identity" -ForegroundColor Green
    } else {
        Write-Host "`nℹ️  No DirSync permissions found for $Type`: $Identity" -ForegroundColor Cyan
    }
}

function Remove-ReadOnlyPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    $acl = Get-Acl "AD:$domainDN"
    $changed = $false
    
    $acesToRemove = $acl.Access | Where-Object {
        ($_.ActiveDirectoryRights -match "ReadProperty") -and
        ($_.InheritanceType -eq "Descendents") -and
        (($_.IdentityReference.Value -eq $accountInfo.SID.Value) -or 
         ($_.IdentityReference.Value -eq $accountInfo.NTAccount.Value)) -and
        ($objectTypes.Values -contains $_.InheritedObjectType)
    }
    
    foreach ($ace in $acesToRemove) {
        $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
        Write-Host "Removing: $($ace.IdentityReference) - ReadProperty on $objectTypeName objects" -ForegroundColor Yellow
        $acl.RemoveAccessRuleSpecific($ace)
        $changed = $true
    }
    
    if ($changed) {
        Set-Acl -Path "AD:$domainDN" -AclObject $acl
        Write-Host "`n✅ Read-Only permissions removed from $Type`: $Identity" -ForegroundColor Green
    } else {
        Write-Host "`nℹ️  No Read-Only permissions found for $Type`: $Identity" -ForegroundColor Cyan
    }
}

function Check-AccountPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    $acl = Get-Acl "AD:$domainDN"
    
    Write-Host "`n=== Permissions Report for: $Identity ===" -ForegroundColor Cyan
    Write-Host "Type: $Type" -ForegroundColor Cyan
    Write-Host "Domain: $domainDN" -ForegroundColor Cyan
    Write-Host "SID: $($accountInfo.SID)" -ForegroundColor Cyan
    Write-Host "NT Account: $($accountInfo.NTAccount)`n" -ForegroundColor Cyan
    
    # Check DirSync permissions
    Write-Host "--- DirSync Permissions ---" -ForegroundColor Yellow
    $dirSyncACEs = $acl.Access | Where-Object {
        (($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)) -and
        (($_.IdentityReference.Value -eq $accountInfo.SID.Value) -or 
         ($_.IdentityReference.Value -eq $accountInfo.NTAccount.Value))
    }
    
    if ($dirSyncACEs) {
        foreach ($ace in $dirSyncACEs) {
            $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
                "DS-Replication-Get-Changes" 
            } else { 
                "DS-Replication-Get-Changes-All" 
            }
            Write-Host "  ✅ $permType" -ForegroundColor Green
        }
    } else {
        Write-Host "  ❌ No DirSync permissions" -ForegroundColor Red
    }
    
    # Check Read-Only permissions
    Write-Host "`n--- Read-Only Permissions ---" -ForegroundColor Yellow
    $readOnlyACEs = $acl.Access | Where-Object {
        ($_.ActiveDirectoryRights -match "ReadProperty") -and
        ($_.InheritanceType -eq "Descendents") -and
        (($_.IdentityReference.Value -eq $accountInfo.SID.Value) -or 
         ($_.IdentityReference.Value -eq $accountInfo.NTAccount.Value))
    }
    
    if ($readOnlyACEs) {
        foreach ($ace in $readOnlyACEs) {
            $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
            if ($objectTypeName) {
                Write-Host "  ✅ ReadProperty on $objectTypeName objects" -ForegroundColor Green
            }
        }
    } else {
        Write-Host "  ❌ No Read-Only permissions" -ForegroundColor Red
    }
    
    Write-Host ""
}

function List-AllPermissions {
    $acl = Get-Acl "AD:$domainDN"
    
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "  COMPLETE PERMISSIONS REPORT" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Domain: $domainDN`n" -ForegroundColor Cyan
    
    # List DirSync Permissions
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
    Write-Host "  DIRSYNC PERMISSIONS" -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
    
    $dirSyncACEs = $acl.Access | Where-Object {
        ($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)
    }
    
    $groupedDirSync = $dirSyncACEs | Group-Object IdentityReference
    
    if ($groupedDirSync) {
        foreach ($group in $groupedDirSync) {
            Write-Host "`nIdentity: $($group.Name)" -ForegroundColor Green
            foreach ($ace in $group.Group) {
                $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
                    "DS-Replication-Get-Changes" 
                } else { 
                    "DS-Replication-Get-Changes-All" 
                }
                Write-Host "  → $permType" -ForegroundColor White
            }
        }
    } else {
        Write-Host "No DirSync permissions found." -ForegroundColor Red
    }
    
    # List Read-Only Permissions
    Write-Host "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
    Write-Host "  READ-ONLY PERMISSIONS" -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
    
    $readOnlyACEs = $acl.Access | Where-Object {
        ($_.ActiveDirectoryRights -match "ReadProperty") -and
        ($_.InheritanceType -eq "Descendents") -and
        ($_.IdentityReference -notlike "*BUILTIN*") -and
        ($_.IdentityReference -notlike "*NT AUTHORITY*") -and
        ($_.IdentityReference -notlike "*Enterprise Domain Controllers*") -and
        ($_.IdentityReference -notlike "*S-1-5-*")
    }
    
    $groupedReadOnly = $readOnlyACEs | Group-Object IdentityReference
    
    if ($groupedReadOnly) {
        foreach ($group in $groupedReadOnly) {
            Write-Host "`nIdentity: $($group.Name)" -ForegroundColor Green
            foreach ($ace in $group.Group) {
                $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
                if ($objectTypeName) {
                    Write-Host "  → ReadProperty on $objectTypeName objects" -ForegroundColor White
                }
            }
        }
    } else {
        Write-Host "No custom Read-Only permissions found." -ForegroundColor Red
    }
    
    Write-Host "`n========================================`n" -ForegroundColor Cyan
}

# Main execution
switch ($Action) {
    "Add-DirSync" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Add-DirSync" -ForegroundColor Red
            exit
        }
        Add-DirSyncPermissions -Identity $Identity -Type $Type
    }
    "Add-ReadOnly" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Add-ReadOnly" -ForegroundColor Red
            exit
        }
        Add-ReadOnlyPermissions -Identity $Identity -Type $Type
    }
    "Remove-DirSync" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Remove-DirSync" -ForegroundColor Red
            exit
        }
        Remove-DirSyncPermissions -Identity $Identity -Type $Type
    }
    "Remove-ReadOnly" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Remove-ReadOnly" -ForegroundColor Red
            exit
        }
        Remove-ReadOnlyPermissions -Identity $Identity -Type $Type
    }
    "Check" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Check" -ForegroundColor Red
            exit
        }
        Check-AccountPermissions -Identity $Identity -Type $Type
    }
    "List" {
        List-AllPermissions
    }
}
```

### Usage Examples:

```powershell
# List all permissions (both DirSync and Read-Only)
.\Manage-ADPermissions.ps1 -Action List

# Add DirSync permissions to user adel.ali
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync

# Add DirSync permissions to computer AALY-LSMH
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-DirSync

# Add Read-Only permissions to user adel.ali
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-ReadOnly

# Add Read-Only permissions to computer AALY-LSMH
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-ReadOnly

# Check what permissions adel.ali has
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check

# Check what permissions computer AALY-LSMH has
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Check

# Remove DirSync permissions from user adel.ali
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Remove-DirSync

# Remove Read-Only permissions from computer AALY-LSMH
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Remove-ReadOnly
```

***

## 2. List All Permissions

### Script: `List-All-Permissions.ps1`

```powershell
<#
.SYNOPSIS
    Lists all DirSync and Read-Only permissions

.DESCRIPTION
    Comprehensive report showing all accounts with DirSync or Read-Only permissions
#>

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$objectTypeNames = @{
    "bf967aba-0de6-11d0-a285-00aa003049e2" = "User"
    "bf967a86-0de6-11d0-a285-00aa003049e2" = "Computer"
    "bf967a9c-0de6-11d0-a285-00aa003049e2" = "Group"
    "5cb41ed0-0e4c-11d0-a286-00aa003049e2" = "Contact"
    "4828cc14-1437-45bc-9b07-ad6f015e5f28" = "InetOrgPerson"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "ForeignSecurityPrincipal"
}

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  COMPLETE PERMISSIONS REPORT" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "Domain: $domainDN`n" -ForegroundColor Cyan

# List DirSync Permissions
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
Write-Host "  DIRSYNC PERMISSIONS" -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow

$dirSyncACEs = $acl.Access | Where-Object {
    ($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)
}

$groupedDirSync = $dirSyncACEs | Group-Object IdentityReference

if ($groupedDirSync) {
    foreach ($group in $groupedDirSync) {
        Write-Host "`nIdentity: $($group.Name)" -ForegroundColor Green
        foreach ($ace in $group.Group) {
            $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
                "DS-Replication-Get-Changes" 
            } else { 
                "DS-Replication-Get-Changes-All" 
            }
            Write-Host "  → $permType" -ForegroundColor White
        }
    }
} else {
    Write-Host "No DirSync permissions found." -ForegroundColor Red
}

# List Read-Only Permissions
Write-Host "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
Write-Host "  READ-ONLY PERMISSIONS" -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow

$readOnlyACEs = $acl.Access | Where-Object {
    ($_.ActiveDirectoryRights -match "ReadProperty") -and
    ($_.InheritanceType -eq "Descendents") -and
    ($_.IdentityReference -notlike "*BUILTIN*") -and
    ($_.IdentityReference -notlike "*NT AUTHORITY*") -and
    ($_.IdentityReference -notlike "*Enterprise Domain Controllers*") -and
    ($_.IdentityReference -notlike "*S-1-5-*")
}

$groupedReadOnly = $readOnlyACEs | Group-Object IdentityReference

if ($groupedReadOnly) {
    foreach ($group in $groupedReadOnly) {
        Write-Host "`nIdentity: $($group.Name)" -ForegroundColor Green
        foreach ($ace in $group.Group) {
            $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
            if ($objectTypeName) {
                Write-Host "  → ReadProperty on $objectTypeName objects" -ForegroundColor White
            }
        }
    }
} else {
    Write-Host "No custom Read-Only permissions found." -ForegroundColor Red
}

Write-Host "`n========================================`n" -ForegroundColor Cyan
```

### Usage:
```powershell
.\List-All-Permissions.ps1
```

***

## 3. Add DirSync Permissions

### For User: `Add-DirSync-User.ps1`

```powershell
<#
.SYNOPSIS
    Grants DirSync permissions to user: adel.ali
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChanges
)
$acl.AddAccessRule($ace1)

$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChangesAll
)
$acl.AddAccessRule($ace2)

Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ DirSync permissions granted to user: $Identity" -ForegroundColor Green
Write-Host "   - DS-Replication-Get-Changes" -ForegroundColor Yellow
Write-Host "   - DS-Replication-Get-Changes-All" -ForegroundColor Yellow
```

### For Computer: `Add-DirSync-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Grants DirSync permissions to computer: AALY-LSMH
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChanges
)
$acl.AddAccessRule($ace1)

$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guidRepChangesAll
)
$acl.AddAccessRule($ace2)

Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ DirSync permissions granted to computer: $Identity" -ForegroundColor Green
Write-Host "   - DS-Replication-Get-Changes" -ForegroundColor Yellow
Write-Host "   - DS-Replication-Get-Changes-All" -ForegroundColor Yellow
```

***

## 4. Add Read-Only Permissions

### For User: `Add-ReadOnly-User.ps1`

```powershell
<#
.SYNOPSIS
    Grants read-only permissions to user: adel.ali
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID
$acl = Get-Acl "AD:$domainDN"

$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

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

Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ Read-Only permissions granted to user: $Identity" -ForegroundColor Green
foreach ($objectType in $objectTypes.Keys) {
    Write-Host "   - ReadProperty on $objectType objects" -ForegroundColor Yellow
}
```

### For Computer: `Add-ReadOnly-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Grants read-only permissions to computer: AALY-LSMH
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID
$acl = Get-Acl "AD:$domainDN"

$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

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

Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "✅ Read-Only permissions granted to computer: $Identity" -ForegroundColor Green
foreach ($objectType in $objectTypes.Keys) {
    Write-Host "   - ReadProperty on $objectType objects" -ForegroundColor Yellow
}
```

***

## 5. Remove DirSync Permissions

### For User: `Remove-DirSync-User.ps1`

```powershell
<#
.SYNOPSIS
    Removes DirSync permissions from user: adel.ali
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID
$ntAccount = $sid.Translate([System.Security.Principal.NTAccount])

$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$changed = $false

$acesToRemove = $acl.Access | Where-Object {
    (($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)) -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value))
}

foreach ($ace in $acesToRemove) {
    $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
        "DS-Replication-Get-Changes" 
    } else { 
        "DS-Replication-Get-Changes-All" 
    }
    Write-Host "Removing: $($ace.IdentityReference) - $permType" -ForegroundColor Yellow
    $acl.RemoveAccessRuleSpecific($ace)
    $changed = $true
}

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "`n✅ DirSync permissions removed from user: $Identity" -ForegroundColor Green
} else {
    Write-Host "`nℹ️  No DirSync permissions found for user: $Identity" -ForegroundColor Cyan
}
```

### For Computer: `Remove-DirSync-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Removes DirSync permissions from computer: AALY-LSMH
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID
$ntAccount = $sid.Translate([System.Security.Principal.NTAccount])

$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$changed = $false

$acesToRemove = $acl.Access | Where-Object {
    (($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)) -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value))
}

foreach ($ace in $acesToRemove) {
    $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
        "DS-Replication-Get-Changes" 
    } else { 
        "DS-Replication-Get-Changes-All" 
    }
    Write-Host "Removing: $($ace.IdentityReference) - $permType" -ForegroundColor Yellow
    $acl.RemoveAccessRuleSpecific($ace)
    $changed = $true
}

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "`n✅ DirSync permissions removed from computer: $Identity" -ForegroundColor Green
} else {
    Write-Host "`nℹ️  No DirSync permissions found for computer: $Identity" -ForegroundColor Cyan
}
```

***

## 6. Remove Read-Only Permissions

### For User: `Remove-ReadOnly-User.ps1`

```powershell
<#
.SYNOPSIS
    Removes read-only permissions from user: adel.ali
#>

Import-Module ActiveDirectory

$Identity = "adel.ali"
$domainDN = (Get-ADDomain).DistinguishedName

$user = Get-ADUser -Identity $Identity -ErrorAction Stop
$sid = $user.SID
$ntAccount = $sid.Translate([System.Security.Principal.NTAccount])

$acl = Get-Acl "AD:$domainDN"

$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

$objectTypeNames = @{
    "bf967aba-0de6-11d0-a285-00aa003049e2" = "User"
    "bf967a86-0de6-11d0-a285-00aa003049e2" = "Computer"
    "bf967a9c-0de6-11d0-a285-00aa003049e2" = "Group"
    "5cb41ed0-0e4c-11d0-a286-00aa003049e2" = "Contact"
    "4828cc14-1437-45bc-9b07-ad6f015e5f28" = "InetOrgPerson"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "ForeignSecurityPrincipal"
}

$changed = $false

$acesToRemove = $acl.Access | Where-Object {
    ($_.ActiveDirectoryRights -match "ReadProperty") -and
    ($_.InheritanceType -eq "Descendents") -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value)) -and
    ($objectTypes.Values -contains $_.InheritedObjectType)
}

foreach ($ace in $acesToRemove) {
    $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
    Write-Host "Removing: $($ace.IdentityReference) - ReadProperty on $objectTypeName objects" -ForegroundColor Yellow
    $acl.RemoveAccessRuleSpecific($ace)
    $changed = $true
}

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "`n✅ Read-Only permissions removed from user: $Identity" -ForegroundColor Green
} else {
    Write-Host "`nℹ️  No Read-Only permissions found for user: $Identity" -ForegroundColor Cyan
}
```

### For Computer: `Remove-ReadOnly-Computer.ps1`

```powershell
<#
.SYNOPSIS
    Removes read-only permissions from computer: AALY-LSMH
#>

Import-Module ActiveDirectory

$Identity = "AALY-LSMH"
$domainDN = (Get-ADDomain).DistinguishedName

$computer = Get-ADComputer -Identity $Identity -ErrorAction Stop
$sid = $computer.SID
$ntAccount = $sid.Translate([System.Security.Principal.NTAccount])

$acl = Get-Acl "AD:$domainDN"

$objectTypes = @{
    "User"                     = [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"
    "Computer"                 = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"
    "Group"                    = [GUID]"bf967a9c-0de6-11d0-a285-00aa003049e2"
    "Contact"                  = [GUID]"5cb41ed0-0e4c-11d0-a286-00aa003049e2"
    "InetOrgPerson"           = [GUID]"4828cc14-1437-45bc-9b07-ad6f015e5f28"
    "ForeignSecurityPrincipal" = [GUID]"89e95b76-444d-4c62-991a-0facbeda640c"
}

$objectTypeNames = @{
    "bf967aba-0de6-11d0-a285-00aa003049e2" = "User"
    "bf967a86-0de6-11d0-a285-00aa003049e2" = "Computer"
    "bf967a9c-0de6-11d0-a285-00aa003049e2" = "Group"
    "5cb41ed0-0e4c-11d0-a286-00aa003049e2" = "Contact"
    "4828cc14-1437-45bc-9b07-ad6f015e5f28" = "InetOrgPerson"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "ForeignSecurityPrincipal"
}

$changed = $false

$acesToRemove = $acl.Access | Where-Object {
    ($_.ActiveDirectoryRights -match "ReadProperty") -and
    ($_.InheritanceType -eq "Descendents") -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value)) -and
    ($objectTypes.Values -contains $_.InheritedObjectType)
}

foreach ($ace in $acesToRemove) {
    $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
    Write-Host "Removing: $($ace.IdentityReference) - ReadProperty on $objectTypeName objects" -ForegroundColor Yellow
    $acl.RemoveAccessRuleSpecific($ace)
    $changed = $true
}

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "`n✅ Read-Only permissions removed from computer: $Identity" -ForegroundColor Green
} else {
    Write-Host "`nℹ️  No Read-Only permissions found for computer: $Identity" -ForegroundColor Cyan
}
```

***

## 7. Check Specific Account Permissions

### Script: `Check-Account-Permissions.ps1`

```powershell
<#
.SYNOPSIS
    Checks all permissions for a specific user or computer

.PARAMETER Identity
    Username or computer name

.PARAMETER Type
    User or Computer

.EXAMPLE
    .\Check-Account-Permissions.ps1 -Identity "adel.ali" -Type User
    .\Check-Account-Permissions.ps1 -Identity "AALY-LSMH" -Type Computer
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$Identity,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet("User","Computer")]
    [string]$Type
)

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

# Get account
if ($Type -eq "User") {
    $obj = Get-ADUser -Identity $Identity -ErrorAction Stop
} else {
    $obj = Get-ADComputer -Identity $Identity -ErrorAction Stop
}

$sid = $obj.SID
$ntAccount = $sid.Translate([System.Security.Principal.NTAccount])

$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$objectTypeNames = @{
    "bf967aba-0de6-11d0-a285-00aa003049e2" = "User"
    "bf967a86-0de6-11d0-a285-00aa003049e2" = "Computer"
    "bf967a9c-0de6-11d0-a285-00aa003049e2" = "Group"
    "5cb41ed0-0e4c-11d0-a286-00aa003049e2" = "Contact"
    "4828cc14-1437-45bc-9b07-ad6f015e5f28" = "InetOrgPerson"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "ForeignSecurityPrincipal"
}

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  PERMISSIONS REPORT" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "Identity: $Identity" -ForegroundColor Cyan
Write-Host "Type: $Type" -ForegroundColor Cyan
Write-Host "Domain: $domainDN" -ForegroundColor Cyan
Write-Host "SID: $sid" -ForegroundColor Cyan
Write-Host "NT Account: $ntAccount`n" -ForegroundColor Cyan

# Check DirSync permissions
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
Write-Host "  DIRSYNC PERMISSIONS" -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow

$dirSyncACEs = $acl.Access | Where-Object {
    (($_.ObjectType -eq $guidRepChanges) -or ($_.ObjectType -eq $guidRepChangesAll)) -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value))
}

if ($dirSyncACEs) {
    foreach ($ace in $dirSyncACEs) {
        $permType = if ($ace.ObjectType -eq $guidRepChanges) { 
            "DS-Replication-Get-Changes" 
        } else { 
            "DS-Replication-Get-Changes-All" 
        }
        Write-Host "  ✅ $permType" -ForegroundColor Green
    }
} else {
    Write-Host "  ❌ No DirSync permissions" -ForegroundColor Red
}

# Check Read-Only permissions
Write-Host "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow
Write-Host "  READ-ONLY PERMISSIONS" -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Yellow

$readOnlyACEs = $acl.Access | Where-Object {
    ($_.ActiveDirectoryRights -match "ReadProperty") -and
    ($_.InheritanceType -eq "Descendents") -and
    (($_.IdentityReference.Value -eq $sid.Value) -or ($_.IdentityReference.Value -eq $ntAccount.Value))
}

if ($readOnlyACEs) {
    foreach ($ace in $readOnlyACEs) {
        $objectTypeName = $objectTypeNames[$ace.InheritedObjectType.ToString()]
        if ($objectTypeName) {
            Write-Host "  ✅ ReadProperty on $objectTypeName objects" -ForegroundColor Green
        }
    }
} else {
    Write-Host "  ❌ No Read-Only permissions" -ForegroundColor Red
}

Write-Host "`n========================================`n" -ForegroundColor Cyan
```

### Usage:
```powershell
# Check user permissions
.\Check-Account-Permissions.ps1 -Identity "adel.ali" -Type User

# Check computer permissions
.\Check-Account-Permissions.ps1 -Identity "AALY-LSMH" -Type Computer
```

***

## Quick Reference

### Permission Types

| Permission | GUID | Purpose | Risk Level |
|------------|------|---------|------------|
| DS-Replication-Get-Changes | 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2 | Read directory changes | Medium [3] |
| DS-Replication-Get-Changes-All | 1131f6ad-9c07-11d1-f79f-00c04fc2dcd2 | Read all changes + passwords | High [3] |
| ReadProperty | N/A | Read object properties | Low [4] |

### Common Tasks

```powershell
# List everything
.\Manage-ADPermissions.ps1 -Action List

# Grant full DirSync to user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync

# Grant read-only to computer
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-ReadOnly

# Check what permissions an account has
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check

# Remove DirSync from user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Remove-DirSync

# Remove read-only from computer
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Remove-ReadOnly
```

***

## Prerequisites

- PowerShell ActiveDirectory Module
- Domain Administrator or Enterprise Administrator privileges
- PowerShell execution policy allowing scripts

```powershell
# Install ActiveDirectory module
Install-WindowsFeature RSAT-AD-PowerShell

# Check module
Import-Module ActiveDirectory
```

***

## Security Best Practices

1. **DirSync Permissions** - Only grant to dedicated service accounts[3][5]
2. **Read-Only Permissions** - Use for monitoring and reporting[4]
3. **Regular Audits** - Run list scripts monthly[1]
4. **Strong Authentication** - Enable MFA for privileged accounts[5]
5. **Least Privilege** - Grant minimum required permissions[1]

***

## Troubleshooting

### Common Issues

**Access Denied:**
- Run PowerShell as Administrator
- Verify Domain Admin rights
- Check ActiveDirectory module is loaded[2]

**Account Not Found:**
- Verify spelling and existence in AD
- For computers, check without $ suffix[3]

**Permissions Not Showing:**
- Wrong script type (DirSync vs Read-Only)
- Use Check script to verify both types
- Wait for AD replication (2-5 minutes)[2]

---

**Document Version:** 2.0  
**Last Updated:** November 11, 2025  
**Tested On:** Windows Server 2016/2019/2022, Active Directory Domainry Domain Services
