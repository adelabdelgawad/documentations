
# 🧩 Active Directory DirSync Permission Management Scripts

These PowerShell scripts allow you to **grant** or **remove** Active Directory “Replicating Directory Changes” permissions for a user (such as a service account or DirSync user).

---

## 🟩 **Grant DirSync Permissions**

```powershell
# Import Active Directory module
Import-Module ActiveDirectory

# Get domain DN
$domainDN = (Get-ADDomain).DistinguishedName

# Get user SID (replace 'adel.ali' with your username if different)
$userSID = (Get-ADUser -Identity "adel.ali").SID

# Get current ACL
$acl = Get-Acl "AD:$domainDN"

# Grant "Replicating Directory Changes" permission
$guid1 = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $userSID,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guid1
)
$acl.AddAccessRule($ace1)

# Grant "Replicating Directory Changes All" (optional but recommended)
$guid2 = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"
$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $userSID,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guid2
)
$acl.AddAccessRule($ace2)

# Apply the changes
Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "`n✅ Replication permissions granted successfully to adel.ali!" -ForegroundColor Green
Write-Host "`nPermissions added:" -ForegroundColor Cyan
Write-Host "  - Replicating Directory Changes" -ForegroundColor Yellow
Write-Host "  - Replicating Directory Changes All" -ForegroundColor Yellow
Write-Host "`nYou can now use DirSync control with this account.`n" -ForegroundColor Green
```

---

## 🟥 **Remove DirSync Permissions**

```powershell
# Import Active Directory module
Import-Module ActiveDirectory

# Get domain DN
$domainDN = (Get-ADDomain).DistinguishedName

# Get user SID
$userSID = (Get-ADUser -Identity "adel.ali").SID

# Get current ACL
$acl = Get-Acl "AD:$domainDN"

# Remove "Replicating Directory Changes" permission
$guid1 = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$ace1 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $userSID,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guid1
)
$acl.RemoveAccessRule($ace1) | Out-Null

# Remove "Replicating Directory Changes All" permission
$guid2 = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"
$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $userSID,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $guid2
)
$acl.RemoveAccessRule($ace2) | Out-Null

# Apply the changes
Set-Acl -Path "AD:$domainDN" -AclObject $acl

Write-Host "`n✅ Replication permissions removed successfully from adel.ali!" -ForegroundColor Green
Write-Host "`nPermissions removed:" -ForegroundColor Cyan
Write-Host "  - Replicating Directory Changes" -ForegroundColor Yellow
Write-Host "  - Replicating Directory Changes All" -ForegroundColor Yellow
Write-Host "`nThe user can no longer use DirSync control.`n" -ForegroundColor Yellow
```

---

## 🔍 **Verify Removal**

```powershell
# Import Active Directory module
Import-Module ActiveDirectory

# Get domain DN
$domainDN = (Get-ADDomain).DistinguishedName

# Get current ACL
$acl = Get-Acl "AD:$domainDN"

# Define GUIDs for DirSync permissions
$replicatingChanges = [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2"
$replicatingChangesAll = [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"

# Search for remaining permissions
$found = $acl.Access | Where-Object {
    ($_.ObjectType -eq $replicatingChanges -or $_.ObjectType -eq $replicatingChangesAll) -and
    $_.IdentityReference -like "*adel.ali*"
}

# Output result
if ($found) {
    Write-Host "`n⚠️  User still has replication permissions:" -ForegroundColor Yellow
    $found | Format-Table IdentityReference, ObjectType, AccessControlType -AutoSize
} else {
    Write-Host "`n✅ No replication permissions found for adel.ali" -ForegroundColor Green
}
```

---

## 🧾 Notes

* Replace **`adel.ali`** with the appropriate service account username.
* Run from an **elevated PowerShell session** with Domain Admin or delegated rights.
* The GUIDs correspond to:

  * `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` → “Replicating Directory Changes”
  * `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2` → “Replicating Directory Changes All”
* Optional third permission (for Azure AD Connect–like setups):
  `1131f6ae-9c07-11d1-f79f-00c04fc2dcd2` → “Replicating Directory Changes in Filtered Set”

---

