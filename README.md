# active-directory-azure-lab
Built a Windows Server 2025 Active Directory domain on Azure, domain controller, OUs, groups, users, Group Policy, and account lifecycle automation with PowerShell.
# Lab 1: Active Directory on Azure

**Windows Server 2025 · Azure Free Account · Identity & Access Management**

I built a full Active Directory environment from scratch on an Azure VM: stood up a domain controller, designed the org structure, provisioned users and groups, enforced security with Group Policy, and ran the day-to-day account lifecycle tasks a help desk handles. Almost all of it done in PowerShell. Below is an overview of what I built, the key commands, and the problems I worked through along the way.

My domain: **`corp.lab`**.

---

## What Active Directory does

Active Directory answers one question for an entire company: *who is allowed to do what?* It's the identity backbone that controls who can log into which machines, who can reach which files, and which policies apply to which people. When someone is hired, IT creates their account and adds them to the right groups. When they leave, disabling one account closes every door at once. The same concepts carry directly into Entra ID (cloud AD) for hybrid environments, which is why this is foundational for IT support, sysadmin, and cloud roles.

---

## What I built

### 1. Azure VM + remote access
Created a free-tier Windows Server 2025 VM (Standard_B2s, East US, RDP open on 3389). Connected from my MacBook using the **Windows App** remote desktop client (the Azure `.rdp` file needs a real RDP client to open on macOS) via the VM's public IP.

### 2. Installed the AD DS role
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```

### 3. Promoted the server to a Domain Controller
Installing the role doesn't create a domain. Promotion does. After a required reboot, I created a new forest:
```powershell
Install-ADDSForest -DomainName "corp.lab" -InstallDns
```
Confirmed with `Get-ADDomain`: domain `corp.lab`, functional level Windows2025, and `testVM.corp.lab` holding all FSMO roles. Live domain controller.

### 4. Built the organisational structure
Created OUs for each department, then security groups for role-based access:
```powershell
New-ADOrganizationalUnit -Name "IT"      -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "HR"      -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "Sales"   -Path "DC=corp,DC=lab"

New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=corp,DC=lab"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=corp,DC=lab"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=corp,DC=lab"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=corp,DC=lab"
```

### 5. Provisioned users and group memberships
Created one user per department and added each to their group (run as a single block so `$password` is defined first):
```powershell
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@corp.lab" `
  -Path "OU=IT,DC=corp,DC=lab" -AccountPassword $password -Enabled $true
# ...bob.patel (Finance), carol.jones (HR), david.smith (Sales) created the same way

Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

### 6. Enforced security with Group Policy
Created an **IT Security Policy** GPO and linked it to the IT OU, enforcing:

| Setting | Value |
|---|---|
| Minimum password length | 12 |
| Password complexity | Enabled |
| Screen lock after inactivity | 900 seconds (15 min) |
| Removable/USB storage | Deny all access |

Verified the link with `Get-GPInheritance -Target "OU=IT,DC=corp,DC=lab"`.

### 7. Ran help-desk account lifecycle tasks
The day-one ticket work, all scriptable in PowerShell:
```powershell
# Reset a password, force change at next login
Set-ADAccountPassword -Identity "bob.patel" -Reset -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true

Unlock-ADAccount   -Identity "carol.jones"   # unlock
Disable-ADAccount  -Identity "david.smith"   # offboard
Enable-ADAccount   -Identity "david.smith"   # re-enable
```

---

## Problems I ran into and solved

| What broke | Cause | Fix |
|---|---|---|
| Mac couldn't open the `.rdp` file | No RDP client on macOS | Installed Windows App, connected by public IP |
| `Install-WindowsFeature -Name GPNC` failed | Wrong feature name | It's `GPMC`; verify with `Get-WindowsFeature -Name *keyword*` |
| Red alerts in Server Manager after installing AD DS | Pending-restart signal, not a failure | Reboot, then continue |
| `Install-ADDSForest` "role change in progress / needs restart" | Pending reboot from the role install | Reboot first, then promote |
| Promotion "failed" by rebooting on its own | The reboot is the success signal | Reconnect, verify with `Get-ADDomain` |
| `New-ADOrganizationalUnit` "server unwilling to process the request" | `-Path` pointed at a nonexistent domain (`DC=lab,DC=local`) | Match the DN to my domain: `DC=corp,DC=lab` |
| `New-ADGroup` "already exists" | Ran the block twice; first run succeeded | Idempotency: check before creating |

---

## Key takeaways

1. **Install is not promote.** Installing the AD DS role doesn't create a domain. You promote the server, with a reboot in between.
2. **Every path must match the real domain.** Most of my errors were a wrong distinguished name. `(Get-ADDomain).DistinguishedName` gives the exact string.
3. **Idempotency matters.** A script safe to run twice checks whether the object already exists first.
4. **Group Policy links to OUs, not groups.** The OU structure is what makes GPOs deliverable.

If I rebuilt this, I'd store the domain DN in a variable up front (`$dn = (Get-ADDomain).DistinguishedName`) and reference `$dn` everywhere instead of retyping the path. That single habit would have prevented most of the errors above.

---

## Verification

```powershell
Get-ADDomain
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName
Get-ADUser -Filter {Enabled -eq $true} | Select Name
Get-ADGroupMember -Identity "IT_Admins"
Get-GPInheritance -Target "OU=IT,DC=corp,DC=lab"
```

**Status: complete.** Azure VM → domain controller → OUs → groups → users → Group Policy → account lifecycle, end to end.
