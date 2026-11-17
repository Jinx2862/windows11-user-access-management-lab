# Windows 11 User Account and Access Management – IT Support Lab

A beginner-friendly, hands-on Windows 11 Home lab demonstrating foundational IT Support and Help Desk tasks involving local user accounts, group membership, User Account Control (UAC), NTFS permissions, and access troubleshooting.

> **Project status:** Practical lab in progress / update this status as you complete and verify each exercise. Only mark a task complete when you have performed it and captured genuine evidence.

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Lab Environment](#lab-environment)
- [Tools and Commands](#tools-and-commands)
- [Project Structure](#project-structure)
- [Exercises](#exercises)
- [Key Troubleshooting Case: Restricted Folder Access](#key-troubleshooting-case-restricted-folder-access)
- [Security and Safety Notes](#security-and-safety-notes)
- [Evidence and Screenshots](#evidence-and-screenshots)
- [What I Learned](#what-i-learned)
- [How to Reproduce the Lab](#how-to-reproduce-the-lab)

## Project Overview

This project simulates common first-line IT Support responsibilities on a Windows 11 computer. It focuses on investigating local accounts, understanding administrative privileges, creating test accounts, configuring folder permissions, and troubleshooting unexpected access.

The lab uses **Windows 11 Home**. Because the Local Users and Groups MMC snap-in (`lusrmgr.msc`) is not included in Windows 11 Home, account investigation and management are performed using built-in PowerShell commands and Windows Settings.

All changes are restricted to the dedicated lab accounts and the `C:\ITSupportLab` directory. Existing personal accounts and Windows system folders should not be modified.

## Objectives

- Enumerate local Windows user accounts.
- Inspect local Administrators and Users group membership.
- Distinguish standard-user permissions from administrator privileges.
- Create dedicated test accounts for controlled permission testing.
- Understand how User Account Control (UAC) requests elevation.
- Configure and inspect NTFS folder permissions.
- Apply the principle of least privilege.
- Investigate and resolve unexpected access to a restricted folder.
- Verify changes and document results as an IT Support ticket / lab report.

## Lab Environment

| Item | Details |
|---|---|
| Operating system | Windows 11 Home |
| Main administration account | `Ajinkya` (local administrator in this lab) |
| Standard test account | `Lab.Standard` |
| Administrator test account | `Lab.Admin` |
| Lab directory | `C:\ITSupportLab` |
| Shell | Windows PowerShell |
| Permission tool | `icacls` |
| Account and group cmdlets | PowerShell `Microsoft.PowerShell.LocalAccounts` module |

**Privacy note:** Replace or redact your computer name and personal account name in public screenshots if you prefer.

## Tools and Commands

### Local account investigation

```powershell
Get-LocalUser
Get-LocalUser | Format-Table Name, Enabled, LastLogon
```

### Local group membership

```powershell
Get-LocalGroupMember -Group "Administrators"
Get-LocalGroupMember -Group "Users"
```

### Create a local test account

Run PowerShell as Administrator. Use a strong, unique lab password and do not publish it.

```powershell
$Password = Read-Host "Enter password for Lab.Standard" -AsSecureString

New-LocalUser -Name "Lab.Standard" `
  -Password $Password `
  -FullName "Lab Standard User" `
  -Description "Standard user account created for IT Support Lab"

Add-LocalGroupMember -Group "Users" -Member "Lab.Standard"
```

Create the administrator test account separately only when completing that exercise:

```powershell
$Password = Read-Host "Enter password for Lab.Admin" -AsSecureString

New-LocalUser -Name "Lab.Admin" `
  -Password $Password `
  -FullName "Lab Administrator" `
  -Description "Administrator test account created for IT Support Lab"

Add-LocalGroupMember -Group "Administrators" -Member "Lab.Admin"
```

### Inspect NTFS permissions

```powershell
icacls "C:\ITSupportLab\Shared"
icacls "C:\ITSupportLab\Restricted"

(Get-Acl "C:\ITSupportLab\Restricted").Access |
  Format-Table IdentityReference, FileSystemRights, AccessControlType, IsInherited -AutoSize
```

### Identify the signed-in account

```powershell
whoami
whoami /groups
```

## Project Structure

The following is the intended repository layout. Adjust it to match the files and screenshots actually included in your repository.

```text
windows11-user-access-management-lab/
│
├── README.md
├── Documentation/
│   ├── Exercise-01-Account-Audit.md
│   └── Exercise-07-Restricted-Folder-Troubleshooting.md
│
├── Screenshots/
│   ├── 01_Local_User_Accounts.png
│   ├── 02_Administrators_Group.png
│   ├── 03_Users_Group.png
│   ├── 11_UAC_Elevation_Test.png
│   └── 36_Final_Project_Folder_Structure.png
│
└── Commands/
    └── README.md
```

You can keep detailed reports and evidence in the repository, or omit any item that you have not yet created. Do not add empty or fabricated evidence.

## Exercises

Update the status column as you complete each exercise.

| Exercise | Task | Status |
|---|---|---|
| 1 | Investigate existing local user accounts and group membership | Completed – reviewed using PowerShell |
| 2 | Create a standard local user (`Lab.Standard`) | Update after verification |
| 3 | Create an administrator test user (`Lab.Admin`) and compare privileges | Update after verification |
| 4 | Investigate sign-in issues and relevant Windows events | Update after completion |
| 5 | Configure and test the `Shared` folder | Update after verification |
| 6 | Configure the `Restricted` folder permissions | Update after verification |
| 7 | Test access restrictions using separate accounts | Troubleshooting in progress / update after retest |
| 8 | Investigate UAC and basic account security settings | Update after verification |
| 9 | Troubleshoot a simulated shared-folder access incident | Update after completion |
| 10 | Perform a final account, group, folder, and permission audit | Update after completion |

## Key Troubleshooting Case: Restricted Folder Access

### Issue

During access testing, the `Lab.Standard` account could modify the `Restricted` folder, although the intended configuration was to prevent that account from accessing it.

### Investigation

The following command was used to inspect the folder's access control list:

```powershell
icacls "C:\ITSupportLab\Restricted"
```

The observed permissions included:

```text
BUILTIN\Administrators:(OI)(CI)(F)
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
BUILTIN\Users:(OI)(CI)(RX)
NT AUTHORITY\Authenticated Users:(OI)(CI)(M)
LAPTOP-AJINKYA\Lab.Admin:(OI)(CI)(F)
```

The `Lab.Standard` account was a member of the local `Users` group. It was also an authenticated Windows user. Therefore, the `Users` and `Authenticated Users` entries granted it access.

### Root Cause

The `Authenticated Users` group had **Modify** permission, and the `Users` group had **Read & Execute** permission. These broad group permissions allowed the standard account to access the folder.

### Intended Resolution

The lab's intended Restricted-folder access policy is:

| Principal | Intended permission |
|---|---|
| `SYSTEM` | Full Control |
| `Administrators` | Full Control |
| `Lab.Admin` | Full Control |
| `Lab.Standard` | No access |
| `Users` | No access |
| `Authenticated Users` | No access |

Before changing permissions, save the current ACL as a record:

```powershell
icacls "C:\ITSupportLab\Restricted" /save `
  "C:\ITSupportLab\Reports\Restricted-ACL-before-fix.txt" /T
```

The following commands remove the unnecessary **Allow** entries for the built-in Users and Authenticated Users groups from the lab's Restricted directory tree:

```powershell
icacls "C:\ITSupportLab\Restricted" /remove:g "*S-1-5-32-545" /T /C
icacls "C:\ITSupportLab\Restricted" /remove:g "*S-1-5-11" /T /C
```

Then inspect the result:

```powershell
icacls "C:\ITSupportLab\Restricted" /T
```

**Important:** Run permission changes only from the administrator account and only against the lab directory. Review the output and verify every file/subfolder. Do not remove `SYSTEM` or `Administrators` access. If the folder contains important data, back it up first.

### Verification

Retest using the separate accounts and record the actual results:

| Account | Folder | Expected result | Actual result |
|---|---|---|---|
| `Lab.Standard` | `Shared` | Can create/modify files if configured | Record test result |
| `Lab.Standard` | `Restricted` | Access denied | Record test result |
| `Lab.Admin` | `Restricted` | Full access | Record test result |

Do not mark the issue resolved until these tests have been performed. If access is still granted, inspect inherited permissions, explicit entries, nested folders, and the account's current sign-in token.

## Security and Safety Notes

- Use dedicated test accounts; do not experiment with your everyday account's membership.
- Never publish passwords, recovery keys, personal documents, or sensitive account information.
- Do not change permissions on `C:\`, `C:\Windows`, `C:\Users`, or folders outside `C:\ITSupportLab`.
- Preserve `SYSTEM` and `Administrators` access to the lab folders.
- Do not disable UAC or weaken Windows security settings to obtain screenshots.
- A UAC secure-desktop prompt may not be capturable using ordinary screenshot tools. If so, document the observed prompt separately and do not claim a screenshot proves it.
- Clearly label simulated incidents and expected results. Record actual test outcomes honestly.

## Evidence and Screenshots

Add genuine screenshots to the `Screenshots/` directory. Suggested evidence includes:

| Screenshot | Evidence |
|---|---|
| `01_Local_User_Accounts.png` | PowerShell output from `Get-LocalUser` |
| `02_Administrators_Group.png` | Administrators group membership |
| `03_Users_Group.png` | Users group membership |
| `11_UAC_Elevation_Test.png` | UAC test evidence; explain if the secure prompt itself could not be captured |
| `36_Final_Project_Folder_Structure.png` | File Explorer showing the final lab folder structure |

Use screenshots from your own Windows environment. Remove or blur personal information where appropriate.

## What I Learned

Complete or revise these points based on your actual work:

- Investigating local Windows accounts with PowerShell.
- Understanding local group membership and administrator privileges.
- Understanding UAC elevation prompts.
- Inspecting NTFS permissions with `icacls` and `Get-Acl`.
- Troubleshooting broad group permissions and inherited ACL entries.
- Applying least privilege to a dedicated test folder.
- Verifying access with separate standard and administrator accounts.
- Documenting technical findings and corrective actions.

## How to Reproduce the Lab

1. Use a Windows 11 test computer or virtual machine.
2. Create a dedicated `C:\ITSupportLab` directory.
3. Inspect existing local accounts and groups using the commands above.
4. Create dedicated test accounts for standard and administrator scenarios.
5. Create `Shared` and `Restricted` folders containing non-sensitive test files.
6. Configure folder permissions only within the lab directory.
7. Sign in separately as each test account and verify access.
8. Record command output, screenshots, and actual results.
9. Review the final ACLs and remove test accounts only after the lab is complete and you no longer need them.

---

**Disclaimer:** This is a personal learning project performed in a controlled Windows environment. It is not an official Microsoft procedure or a production access-control recommendation. Always validate permissions and test changes before applying them in a business environment.
