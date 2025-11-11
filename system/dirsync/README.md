---

## Quick reference — GUIDs used

* `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` → **Replicating Directory Changes**
* `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2` → **Replicating Directory Changes All**
* `1131f6ae-9c07-11d1-f79f-00c04fc2dcd2` → *(optional)* **Replicating Directory Changes in Filtered Set**

---

## 1) List Current DirSync Permissions

```powershell
<#
Usage:
  Save as List-DirSync.ps1 and run in elevated PowerShell.

Purpose:
  Lists all ACEs on the domain root that grant DirSync-related extended rights.
#>

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges     = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll  = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepFilteredSet = [GUID]"1131f6ae-9c07-11d1-f79f-00c04fc2dcd2"

$results = $acl.Access |
    Where-Object { $_.ObjectType -in @($guidRepChanges, $guidRepChangesAll, $guidRepFilteredSet) } |
    Select-Object @{Name='Account';Expression={$_.IdentityReference}},
                  @{Name='Right';Expression={
                      switch ($_.ObjectType.Guid.ToString()) {
                        $guidRepChanges.Guid.ToString()     { 'Replicating Directory Changes' }
                        $guidRepChangesAll.Guid.ToString()  { 'Replicating Directory Changes All' }
                        $guidRepFilteredSet.Guid.ToString() { 'Replicating Directory Changes in Filtered Set' }
                        default { $_.ObjectType }
                      }
                  }},
                  AccessControlType,
                  IsInherited,
                  InheritanceType,
                  ActiveDirectoryRights

if ($results) {
    $results | Format-Table -AutoSize
} else {
    Write-Host "No DirSync replication permissions found on domain root." -ForegroundColor Yellow
}
```

---

## 2) Add DirSync (Read-Only) for a **User** Account

```powershell
<#
Usage:
  .\Add-DirSync-User.ps1 -Username "svc-dirsync"
  (Use sAMAccountName or UPN)

Purpose:
  Grants Replicating Directory Changes and Replicating Directory Changes All
  to a user. Safe: checks existing ACEs and only adds missing ones.
#>

param(
    [Parameter(Mandatory=$true)] [string]$Username
)

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$user = Get-ADUser -Identity $Username -ErrorAction Stop
$userSID = $user.SID

$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

function Add-IfMissing {
    param($acl, $sid, $guid)
    $exists = $acl.Access | Where-Object {
        ($_.ObjectType -eq $guid) -and ($_.IdentityReference -eq $sid)
    }
    if (-not $exists) {
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $sid,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guid
        )
        $acl.AddAccessRule($ace)
        return $true
    }
    return $false
}

$changed = $false
if (Add-IfMissing -acl $acl -sid $userSID -guid $guidRepChanges) { $changed = $true }
if (Add-IfMissing -acl $acl -sid $userSID -guid $guidRepChangesAll) { $changed = $true }

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "✅ DirSync permissions granted to user $Username" -ForegroundColor Green
} else {
    Write-Host "ℹ️  User $Username already has DirSync permissions (no changes made)." -ForegroundColor Cyan
}
```

---

## 3) Add DirSync (Read-Only) for a **Computer** Account

```powershell
<#
Usage:
  .\Add-DirSync-Computer.ps1 -ComputerName "WEB01"
  (Use computer sAMAccountName without trailing $; Get-ADComputer accepts either.)

Purpose:
  Grants Replicating Directory Changes and Replicating Directory Changes All
  to a machine account (e.g., hostname$). Checks existing ACEs first.
#>

param(
    [Parameter(Mandatory=$true)] [string]$ComputerName
)

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName
$computer = Get-ADComputer -Identity $ComputerName -ErrorAction Stop
$computerSID = $computer.SID

$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

function Add-IfMissing {
    param($acl, $sid, $guid)
    $exists = $acl.Access | Where-Object {
        ($_.ObjectType -eq $guid) -and ($_.IdentityReference -eq $sid)
    }
    if (-not $exists) {
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $sid,
            [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
            [System.Security.AccessControl.AccessControlType]::Allow,
            $guid
        )
        $acl.AddAccessRule($ace)
        return $true
    }
    return $false
}

$changed = $false
if (Add-IfMissing -acl $acl -sid $computerSID -guid $guidRepChanges) { $changed = $true }
if (Add-IfMissing -acl $acl -sid $computerSID -guid $guidRepChangesAll) { $changed = $true }

if ($changed) {
    Set-Acl -Path "AD:$domainDN" -AclObject $acl
    Write-Host "✅ DirSync permissions granted to computer $ComputerName$ (machine account)" -ForegroundColor Green
} else {
    Write-Host "ℹ️  Computer $ComputerName already has DirSync permissions (no changes made)." -ForegroundColor Cyan
}
```

---

## 4) Remove DirSync Permissions (User or Computer)

```powershell
<#
Usage:
  .\Remove-DirSync.ps1 -Identity "svc-dirsync" -Type User
  .\Remove-DirSync.ps1 -Identity "WEB01" -Type Computer

Purpose:
  Removes the two DirSync ACEs for a specified user or computer account.
  It looks up the account SID and removes any matching ACEs.
#>

param(
    [Parameter(Mandatory=$true)] [string]$Identity,
    [Parameter(Mandatory=$true)] [ValidateSet("User","Computer")] [string]$Type
)

Import-Module ActiveDirectory

$domainDN = (Get-ADDomain).DistinguishedName

if ($Type -eq 'User') {
    $obj = Get-ADUser -Identity $Identity -ErrorAction Stop
} else {
    $obj = Get-ADComputer -Identity $Identity -ErrorAction Stop
}
$sid = $obj.SID

$acl = Get-Acl "AD:$domainDN"

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
    Write-Host "✅ DirSync permissions removed for $Type $Identity" -ForegroundColor Green
} else {
    Write-Host "ℹ️  No DirSync permissions found for $Type $Identity (no changes made)." -ForegroundColor Cyan
}
```

---

## 5) Verify (Quick Check)

```powershell
# Example: check for 'svc-dirsync'
Import-Module ActiveDirectory
$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:$domainDN"

$guidRepChanges    = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$guidRepChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

$search = $acl.Access | Where-Object {
    ($_.ObjectType -in @($guidRepChanges, $guidRepChangesAll)) -and
    ($_.IdentityReference -like "*svc-dirsync*" -or $_.IdentityReference -like "*WEB01*")
}
$search | Format-Table IdentityReference, ObjectType, AccessControlType -AutoSize
```

---

## Best practices & notes

* Always run in an **elevated** PowerShell session with proper delegation.
* Prefer giving DirSync rights to a **dedicated service account** or **machine account** used strictly for read-only synchronization.
* Consider adding `Replicating Directory Changes in Filtered Set` (`1131f6ae...`) only if required by your sync design (e.g., filtered replication or hybrid/AAD Connect scenarios).
* Audit and document which accounts hold these rights. Periodically review and revoke unused entries.
* When giving rights to computer accounts, ensure the service runs under the machine identity (Kerberos keytab or proper domain-joined service) and that you use LDAPS or secure Kerberos-authenticated LDAP (SASL/GSSAPI).

---
