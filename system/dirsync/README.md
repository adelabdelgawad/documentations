# Active Directory DirSync & Read-Only Permissions Management Guide

Complete guide for managing Directory Synchronization (DirSync) and Read-Only permissions in Active Directory for both users and computers.

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
2. [Validate DirSync Permissions](#2-validate-dirsync-permissions)
3. [Add DirSync Permissions](#3-add-dirsync-permissions)
4. [Add Read-Only Permissions](#4-add-read-only-permissions)
5. [Remove DirSync Permissions](#5-remove-dirsync-permissions)
6. [Remove Read-Only Permissions](#6-remove-read-only-permissions)
7. [Check Specific Account Permissions](#7-check-specific-account-permissions)
8. [List All Permissions](#8-list-all-permissions)

***

## 1. Universal Management Scripts

### Master Permission Manager

This comprehensive script manages both DirSync and Read-Only permissions for users and computers.

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
    Add-DirSync, Add-ReadOnly, Remove-DirSync, Remove-ReadOnly, Check, List, Validate-DirSync
    
.EXAMPLE
    .\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync
    .\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-ReadOnly
    .\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check
    .\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Validate-DirSync
    .\Manage-ADPermissions.ps1 -Action List
#>

param(
    [Parameter(Mandatory=$false)]
    [string]$Identity,
    
    [Parameter(Mandatory=$false)]
    [ValidateSet("User","Computer")]
    [string]$Type,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet("Add-DirSync","Add-ReadOnly","Remove-DirSync","Remove-ReadOnly","Check","List","Validate-DirSync")]
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

function Validate-DirSyncPermissions {
    param($Identity, $Type)
    
    Write-Host "`n=== Validating DirSync Permissions ===" -ForegroundColor Cyan
    Write-Host "Checking $Type`: $Identity`n" -ForegroundColor White
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { 
        exit 1 
    }
    
    $userSID = $accountInfo.SID
    
    Write-Host "$Type Found: $($accountInfo.Object.DistinguishedName)" -ForegroundColor Gray
    Write-Host "SID: $userSID" -ForegroundColor Gray
    Write-Host "Domain: $domainDN`n" -ForegroundColor Gray
    
    # Get domain ACL
    try {
        $acl = Get-Acl "AD:$domainDN"
    }
    catch {
        Write-Host "[✗] ERROR: Failed to retrieve domain ACL" -ForegroundColor Red
        exit 1
    }
    
    # Check for permissions
    $hasDirSync = $false
    $hasDirSyncAll = $false
    
    foreach ($ace in $acl.Access) {
        try {
            $aceIdentity = $ace.IdentityReference.Translate([System.Security.Principal.SecurityIdentifier])
            
            if ($aceIdentity -eq $userSID) {
                if ($ace.ObjectType -eq $guidRepChanges) {
                    $hasDirSync = $true
                    Write-Host "[✓] Found: DS-Replication-Get-Changes (Replicating Directory Changes)" -ForegroundColor Green
                }
                if ($ace.ObjectType -eq $guidRepChangesAll) {
                    $hasDirSyncAll = $true
                    Write-Host "[✓] Found: DS-Replication-Get-Changes-All (Replicating Directory Changes All)" -ForegroundColor Green
                }
            }
        }
        catch {
            # Skip ACEs that can't be translated
            continue
        }
    }
    
    # Report missing permissions
    Write-Host ""
    if (-not $hasDirSync) {
        Write-Host "[✗] MISSING: DS-Replication-Get-Changes" -ForegroundColor Red
    }
    if (-not $hasDirSyncAll) {
        Write-Host "[✗] MISSING: DS-Replication-Get-Changes-All" -ForegroundColor Red
    }
    
    # Final status
    Write-Host "`n=== VALIDATION RESULT ===" -ForegroundColor Yellow
    if ($hasDirSync -and $hasDirSyncAll) {
        Write-Host "[✓] SUCCESS: $Type has all required DirSync permissions" -ForegroundColor Green
        Write-Host "`nThe $Type can perform DirSync operations.`n" -ForegroundColor Gray
        exit 0
    }
    else {
        Write-Host "[✗] FAILED: $Type is missing DirSync permissions" -ForegroundColor Red
        Write-Host "`nTo grant permissions, run:" -ForegroundColor Yellow
        Write-Host ".\Manage-ADPermissions.ps1 -Identity '$Identity' -Type $Type -Action Add-DirSync" -ForegroundColor Yellow
        Write-Host "Note: Domain Admin privileges required.`n" -ForegroundColor Gray
        exit 1
    }
}

function Add-DirSyncPermissions {
    param($Identity, $Type)
    
    $accountInfo = Get-AccountInfo -Identity $Identity -Type $Type
    if (-not $accountInfo.Success) { return }
    
    Write-Host "`n=== Granting DirSync Permissions ===" -ForegroundColor Cyan
    Write-Host "Target $Type`: $Identity`n" -ForegroundColor White
    
    $acl = Get-Acl "AD:$domainDN"
    
    # Check if permissions already exist
    $hasDirSync = $false
    $hasDirSyncAll = $false
    
    foreach ($ace in $acl.Access) {
        try {
            $aceIdentity = $ace.IdentityReference.Translate([System.Security.Principal.SecurityIdentifier])
            if ($aceIdentity -eq $accountInfo.SID) {
                if ($ace.ObjectType -eq $guidRepChanges) {
                    $hasDirSync = $true
                }
                if ($ace.ObjectType -eq $guidRepChangesAll) {
                    $hasDirSyncAll = $true
                }
            }
        }
        catch {
            continue
        }
    }
    
    # Add DS-Replication-Get-Changes if not exists
    if (-not $hasDirSync) {
        Write-Host "Granting: DS-Replication-Get-Changes..." -ForegroundColor Yellow
        $ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $accountInfo.SID,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guidRepChanges
        )
        $acl.AddAccessRule($ace1)
        Write-Host "[✓] Permission added" -ForegroundColor Green
    } else {
        Write-Host "[✓] Already has: DS-Replication-Get-Changes" -ForegroundColor Green
    }
    
    # Add DS-Replication-Get-Changes-All if not exists
    if (-not $hasDirSyncAll) {
        Write-Host "Granting: DS-Replication-Get-Changes-All..." -ForegroundColor Yellow
        $ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $accountInfo.SID,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guidRepChangesAll
        )
        $acl.AddAccessRule($ace2)
        Write-Host "[✓] Permission added" -ForegroundColor Green
    } else {
        Write-Host "[✓] Already has: DS-Replication-Get-Changes-All" -ForegroundColor Green
    }
    
    # Apply the ACL if changes were made
    if (-not $hasDirSync -or -not $hasDirSyncAll) {
        try {
            Write-Host "`nApplying permissions to domain..." -ForegroundColor Yellow
            Set-Acl "AD:$domainDN" -AclObject $acl
            Write-Host "[✓] Permissions applied successfully!`n" -ForegroundColor Green
        }
        catch {
            Write-Host "[ERROR] Failed to apply permissions: $($_.Exception.Message)" -ForegroundColor Red
            exit 1
        }
    } else {
        Write-Host "`n[✓] No changes needed - $Type already has all required permissions`n" -ForegroundColor Green
    }
    
    Write-Host "=== Summary ===" -ForegroundColor Cyan
    Write-Host "$Type`: $Identity" -ForegroundColor White
    Write-Host "Permissions: DS-Replication-Get-Changes + DS-Replication-Get-Changes-All" -ForegroundColor White
    Write-Host "Status: GRANTED" -ForegroundColor Green
    Write-Host "`nNote: These permissions allow reading all directory data for synchronization." -ForegroundColor Gray
    Write-Host "      They do NOT allow modifying objects.`n" -ForegroundColor Gray
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
    "Validate-DirSync" {
        if (-not $Identity -or -not $Type) {
            Write-Host "❌ Error: -Identity and -Type are required for Validate-DirSync" -ForegroundColor Red
            exit
        }
        Validate-DirSyncPermissions -Identity $Identity -Type $Type
    }
    "List" {
        List-AllPermissions
    }
}
```

### Usage Examples:

```powershell
# Validate DirSync permissions for user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Validate-DirSync

# Validate DirSync permissions for computer
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Validate-DirSync

# List all permissions (both DirSync and Read-Only)
.\Manage-ADPermissions.ps1 -Action List

# Add DirSync permissions to user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync

# Add DirSync permissions to computer
.\Manage-ADPermissions.ps1 -Identity "AALY-LSMH" -Type Computer -Action Add-DirSync

# Check what permissions a user has
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check

# Remove DirSync permissions from user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Remove-DirSync
```

***

## 2. Validate DirSync Permissions

### Standalone Validation Script: `Validate-DirSync-Permissions.ps1`

```powershell
<#
.SYNOPSIS
    Validates DirSync permissions for a user or computer

.DESCRIPTION
    Checks if the specified account has both "Replicating Directory Changes" and 
    "Replicating Directory Changes All" permissions at the domain root.

.PARAMETER Identity
    The username or computer name

.PARAMETER Type
    User or Computer

.EXAMPLE
    .\Validate-DirSync-Permissions.ps1 -Identity "adel.ali" -Type User
    .\Validate-DirSync-Permissions.ps1 -Identity "AALY-LSMH" -Type Computer

.OUTPUTS
    Exit code 0: Has all required permissions
    Exit code 1: Missing permissions or error
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$Identity,
    
    [Parameter(Mandatory=$true)]
    [ValidateSet("User","Computer")]
    [string]$Type
)

Import-Module ActiveDirectory

Write-Host "`n=== Validating DirSync Permissions ===" -ForegroundColor Cyan
Write-Host "Checking $Type`: $Identity`n" -ForegroundColor White

# Get domain DN and account
try {
    $domainDN = (Get-ADDomain).DistinguishedName
    
    if ($Type -eq "User") {
        $accountObj = Get-ADUser -Identity $Identity -ErrorAction Stop
    } else {
        $accountObj = Get-ADComputer -Identity $Identity -ErrorAction Stop
    }
    
    $userSID = $accountObj.SID
    
    Write-Host "$Type Found: $($accountObj.DistinguishedName)" -ForegroundColor Gray
    Write-Host "SID: $userSID" -ForegroundColor Gray
    Write-Host "Domain: $domainDN`n" -ForegroundColor Gray
}
catch {
    Write-Host "[✗] ERROR: $Type '$Identity' not found" -ForegroundColor Red
    Write-Host "    $($_.Exception.Message)" -ForegroundColor Red
    exit 1
}

# Get domain ACL
try {
    $acl = Get-Acl "AD:$domainDN"
}
catch {
    Write-Host "[✗] ERROR: Failed to retrieve domain ACL" -ForegroundColor Red
    Write-Host "    Ensure you have appropriate permissions" -ForegroundColor Red
    exit 1
}

# DirSync Control GUIDs
$dirSyncGuid = "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$dirSyncAllGuid = "1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

# Check for permissions
$hasDirSync = $false
$hasDirSyncAll = $false

foreach ($ace in $acl.Access) {
    try {
        $aceIdentity = $ace.IdentityReference.Translate([System.Security.Principal.SecurityIdentifier])
        
        if ($aceIdentity -eq $userSID) {
            if ($ace.ObjectType.Guid -eq $dirSyncGuid) {
                $hasDirSync = $true
                Write-Host "[✓] Found: DS-Replication-Get-Changes (Replicating Directory Changes)" -ForegroundColor Green
            }
            if ($ace.ObjectType.Guid -eq $dirSyncAllGuid) {
                $hasDirSyncAll = $true
                Write-Host "[✓] Found: DS-Replication-Get-Changes-All (Replicating Directory Changes All)" -ForegroundColor Green
            }
        }
    }
    catch {
        # Skip ACEs that can't be translated (orphaned SIDs, etc.)
        continue
    }
}

# Report missing permissions
Write-Host ""
if (-not $hasDirSync) {
    Write-Host "[✗] MISSING: DS-Replication-Get-Changes" -ForegroundColor Red
}
if (-not $hasDirSyncAll) {
    Write-Host "[✗] MISSING: DS-Replication-Get-Changes-All" -ForegroundColor Red
}

# Final status
Write-Host "`n=== VALIDATION RESULT ===" -ForegroundColor Yellow
if ($hasDirSync -and $hasDirSyncAll) {
    Write-Host "[✓] SUCCESS: $Type has all required DirSync permissions" -ForegroundColor Green
    Write-Host "`nThe $Type can perform DirSync operations.`n" -ForegroundColor Gray
    exit 0
}
else {
    Write-Host "[✗] FAILED: $Type is missing DirSync permissions" -ForegroundColor Red
    Write-Host "`nTo grant permissions, run:" -ForegroundColor Yellow
    Write-Host ".\Manage-ADPermissions.ps1 -Identity '$Identity' -Type $Type -Action Add-DirSync" -ForegroundColor Yellow
    Write-Host "Note: Domain Admin privileges required.`n" -ForegroundColor Gray
    exit 1
}
```

### Quick One-Liner Validation

```powershell
# Validate user
$user = "adel.ali"
$sid = (Get-ADUser $user).SID
$acl = Get-Acl "AD:$((Get-ADDomain).DistinguishedName)"
$result = $acl.Access | Where-Object { 
    $_.IdentityReference.Translate([System.Security.Principal.SecurityIdentifier]) -eq $sid -and 
    ($_.ObjectType.Guid -eq "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2" -or 
     $_.ObjectType.Guid -eq "1131f6ad-9c07-11d1-f79f-00c04fc2dcd2")
}
if ($result.Count -eq 2) { Write-Host "✓ Has DirSync" -ForegroundColor Green } else { Write-Host "✗ Missing DirSync" -ForegroundColor Red }
```

***

## Quick Reference

### Permission Types

| Permission | GUID | Purpose | Risk Level |
|------------|------|---------|------------|
| DS-Replication-Get-Changes | 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2 | Read directory changes | Medium |
| DS-Replication-Get-Changes-All | 1131f6ad-9c07-11d1-f79f-00c04fc2dcd2 | Read all changes + passwords | High |
| ReadProperty | N/A | Read object properties | Low |

### Common Tasks

```powershell
# Validate DirSync permissions
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Validate-DirSync

# List everything
.\Manage-ADPermissions.ps1 -Action List

# Grant full DirSync to user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Add-DirSync

# Check what permissions an account has
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Check

# Remove DirSync from user
.\Manage-ADPermissions.ps1 -Identity "adel.ali" -Type User -Action Remove-DirSync
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

1. **DirSync Permissions** - Only grant to dedicated service accounts
2. **Read-Only Permissions** - Use for monitoring and reporting
3. **Regular Audits** - Run validation scripts monthly
4. **Strong Authentication** - Enable MFA for privileged accounts
5. **Least Privilege** - Grant minimum required permissions

***

**Document Version:** 3.0  
**Last Updated:** November 13, 2025  
**Tested On:** Windows Server 2016/2019/2022, Active Directory Domain Services
