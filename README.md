# 🏛️ Active Directory DS, GPO Security & FSRM Storage Management

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Category: Windows Server](https://img.shields.io/badge/Category-Windows%20Server%202022-0078D4?logo=windows)](https://obkworks.tr)
[![Topic: MCSA Lab](https://img.shields.io/badge/Lab-MCSA%20Practical-0A66C2)](https://obkworks.tr)

> Practical enterprise infrastructure lab deploying Active Directory Domain Services (AD DS), structured AGDLP Organizational Units, Group Policy (GPO) security hardening, SMB/NTFS permissions, and File Server Resource Manager (FSRM) quota & file screening.

---

## ℹ️ Overview

In this lab scenario, **Active Directory Domain Services (AD DS)** is deployed on Windows Server 2022 (`obk.local`) following the AGDLP access control framework. Centralized domain administration is enforced using targeted Group Policy Objects (GPOs) for password complexity, environment restrictions, and local administrator protection. File share management is implemented with synchronized SMB/NTFS permissions, 2GB hard quotas, and automated executable file screening via **File Server Resource Manager (FSRM)**.

---

## 📐 1. Network Topology & Host Architecture

Lab environment servers and workstations setup for domain `obk.local`:

| Hostname      | Domain Role                                  | Operating System      | IP Address      | Subnet Mask     | Preferred DNS   | Default Gateway |
| :------------ | :------------------------------------------- | :-------------------- | :-------------- | :-------------- | :-------------- | :-------------- |
| **DC1**       | Primary Domain Controller (PDC) / DNS / FSRM | Windows Server 2022   | `192.168.10.10` | `255.255.255.0` | `127.0.0.1`     | `192.168.10.1`  |
| **Client-01** | Domain Member Workstation                    | Windows 10 Enterprise | `192.168.10.50` | `255.255.255.0` | `192.168.10.10` | `192.168.10.1`  |

---

## 🧙‍♂️ 2. Step-by-Step AD DS Promotion Wizard

### Phase 1: Add Roles and Features Wizard

1. Open **Server Manager** and click on **Manage > Add Roles and Features**.
2. On the **Before You Begin** page, click **Next**.
3. Select **Role-based or feature-based installation** and click **Next**.
4. Select `DC1` server from the Server Pool and click **Next**.
5. Check **Active Directory Domain Services**. Click **Add Features** in the pop-up window.
6. In the **Features** page, ensure **Group Policy Management** is selected and click **Next**.
7. Review the AD DS summary information and click **Next**.
8. On the **Confirmation** page, click **Install** to begin role installation.

### Phase 2: Domain Controller Promotion Wizard

1. Click the notification flag icon at the top of Server Manager and select **Promote this server to a domain controller**.
2. Select **Add a new forest** and enter `obk.local` as the Root domain name.
3. Set Functional Level to **Windows Server 2016**. Keep **DNS** and **Global Catalog (GC)** checked, then enter the DSRM password.
4. In **DNS Options**, ignore the delegation warning and click **Next**.
5. Confirm the suggested NetBIOS name (`OBK`) and click **Next**.
6. Leave default paths for Database (`C:\Windows\NTDS`), Log files, and SYSVOL intact, then click **Next**.
7. After passing the Prerequisites Check, click **Install**. The server will automatically restart upon completion.

---

## 🏢 3. Organizational Unit (OU) & Group Hierarchy (AGDLP)

Logical OU structure designed for granular GPO scoping and security group management based on the **AGDLP** (Account -> Global -> Domain Local -> Permission) standard.

### Automated Infrastructure Provisioning (PowerShell)

```powershell
# =========================================================
# 1. OU Provisioning Script
# =========================================================
Import-Module ActiveDirectory

New-ADOrganizationalUnit -Name "Workstations" -Path "DC=obk,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Departments" -Path "DC=obk,DC=local" -ProtectedFromAccidentalDeletion $true

$departments = @("HR", "IT", "Sales")
foreach ($dept in$departments) {
    New-ADOrganizationalUnit -Name $dept -Path "OU=Departments,DC=obk,DC=local" -ProtectedFromAccidentalDeletion $true
}

# =========================================================
# 2. Security Group Provisioning Script
# =========================================================
New-ADGroup -Name "GG_HR_Dept" -GroupScope Global -GroupCategory Security -Path "OU=HR,OU=Departments,DC=obk,DC=local"
New-ADGroup -Name "GG_IT_Dept" -GroupScope Global -GroupCategory Security -Path "OU=IT,OU=Departments,DC=obk,DC=local"
New-ADGroup -Name "GG_Sales_Dept" -GroupScope Global -GroupCategory Security -Path "OU=Sales,OU=Departments,DC=obk,DC=local"

# =========================================================
# 3. User Creation & Group Membership Script
# =========================================================
$securePassword = ConvertTo-SecureString "P@ssw0rd2026!" -AsPlainText -Force

New-ADUser -Name "user.hr" -SamAccountName "user.hr" `
    -UserPrincipalName "user.hr@obk.local" `
    -Path "OU=HR,OU=Departments,DC=obk,DC=local" `
    -AccountPassword $securePassword -Enabled $true `
    -ChangePasswordAtLogon $false

Add-ADGroupMember -Identity "GG_HR_Dept" -Members "user.hr"
```

---

## 🛡️ 4. Group Policy Objects (GPO) Configuration

Group policies implemented for domain-wide centralized security hardening and user environment standardization:

| GPO Name                           | Applied Target              | Description & Policy Settings                                                                                                                                   |
| :--------------------------------- | :-------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Default Domain Policy**          | `obk.local` (Entire Domain) | **Password & Lockout:** Enforces password complexity (min 10 characters) and automatic account lockout after 3 failed attempts to mitigate Brute-Force attacks. |
| **Security Hardening Policy**      | `OU=Workstations`           | **System Hardening:** Disables insecure legacy protocols (LM/NTLMv1), blocks execution of unknown binaries in Temp directories, and controls local privileges.  |
| **User Environment Policy**        | `OU=Departments`            | **Desktop Experience:** Sets corporate wallpaper centrally, enforces interactive logon banner, and restricts access to Control Panel.                           |
| **Restricted Local Admins Policy** | `OU=Workstations`           | **Privilege Management:** Removes standard domain users from local computer Administrators group to prevent unauthorized local system changes.                  |

---

## 💾 5. FSRM & File Share Provisioning Wizard

Phase 1: Installing FSRM Role

1. Open Server Manager and navigate to Add Roles and Features.
2. Under Server Roles, expand File and Storage Services > File and iSCSI Services, then check File Server Resource Manager.
3. Click Add Features and complete the wizard installation.

Phase 2: SMB Share & NTFS Alignment

1. Create storage folder C:\Shares\HR_Data on DC1.
2. Configure SMB Share permissions: Grant GG_HR_Dept Change permission.
3. Configure NTFS permissions (Security tab): Disable inheritance, explicit remove Users, and grant GG_HR_Dept Modify permission.

Phase 3: Quota & File Screening Configuration

1. Open File Server Resource Manager (fsrm.msc).
2. Under Quota Management, create a 2 GB Hard Quota on path C:\Shares\HR_Data.
3. Under File Screen Management, apply Block Executable Files template to prevent .exe, .bat, .vbs, and .cmd files from being stored.

## Automated Share & FSRM Configuration Script (PowerShell)

### Folder Share, NTFS Permissions & FSRM Setup

### Step 1: Create Directory and SMB Share

```
New-Item -Path "C:\Shares\HR_Data" -ItemType Directory -Force
New-SmbShare -Name "HR_Data" -Path "C:\Shares\HR_Data" -FullAccess "Administrators" -ChangeAccess "OBK\GG_HR_Dept"
```

### Step 2: Configure NTFS Permissions

```
$acl = Get-Acl "C:\Shares\HR_Data"
$acl.SetAccessRuleProtection($true,$false) # Disable inheritance, remove inherited rules
$adminRule = New-Object System.Security.AccessControl.FileSystemAccessRule("Administrators", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$hrRule = New-Object System.Security.AccessControl.FileSystemAccessRule("OBK\GG_HR_Dept", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($adminRule)
$acl.AddAccessRule($hrRule)
Set-Acl -Path "C:\Shares\HR_Data" -AclObject $acl
```

### Step 3: Configure FSRM Quota & File Screening

```
Install-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools
New-FsrmQuota -Path "C:\Shares\HR_Data" -Template "2 GB Hard Quota"
New-FsrmFileScreen -Path "C:\Shares\HR_Data" -Template "Block Executable Files"
```

---

## 🔍 6. Client Domain Join & Verification Commands

Client Joining Domain Steps

1. Set Preferred DNS on Client-01 network adapter to 192.168.10.10 (DC1 IP).
2. Open System Properties > Computer Name, change Member Of to Domain: obk.local.
3. Enter Domain Administrator credentials and reboot the computer when prompted.
4. Log in using OBK\user.hr to verify GPO restrictions and share connectivity.

Verification CLI Commands
Run these commands on Client-01 or DC1 to verify active domain health and security rules:

```
:: Force Group Policy update on Client workstation
gpupdate /force

:: Display applied GPOs for the logged-in user and computer
gpresult /r

:: Query FSMO Roles from Domain Controller
netdom query fsmo

:: Verify active SMB network connections
net use
```

### Verify FSRM Quotas and File Screen Status

```
Get-FsrmQuota -Path "C:\Shares\HR_Data"
Get-FsrmFileScreen -Path "C:\Shares\HR_Data"
```

---

## ⚠️ 7. Common Pitfalls & Troubleshooting

1. DNS Resolution Failure During Domain Join: If Client-01 fails to locate the domain controller, verify that the primary DNS address on the client's network adapter points strictly to 192.168.10.10 and not a external public DNS router.

2. SMB vs NTFS Permission Conflicts: Users will be restricted by whichever permission layer is more restrictive. Ensure SMB Share permissions (Change) match or exceed NTFS permissions (Modify).

3. SYSVOL GPO Replication Delay: If newly created GPOs do not take effect immediately, run gpupdate /force on the target computer or check dfsrdiag /testreplication if multiple Domain Controllers exist.

---

## 🏁 8. Conclusion

1. Deploying Active Directory DS alongside GPO and FSRM provides a centralized security framework for user authentication, privilege boundary enforcement, and enterprise file storage management.

2. Aligning AGDLP security group standards with strict NTFS access limits and automated FSRM executable file screening significantly lowers unauthorized lateral movement and ransomware propagation risks.

🔗 Live Portfolio: https://obkworks.tr

📄 License: MIT License — © 2026 obkWorks
