# Lab 02 — Group Policy and file permissions

[← Back to overview](../README.md) · [Previous lab](01-windows-domain.md) · [Command cheat sheet](../CHEATSHEET.md)

With the domain running, I used Lab 02 to configure user policies and file access. Everything was done on the same DC01 and CLIENT01 VMs.

## OUs and groups

I created departmental organizational units and moved the existing Windows 11 computer into its own OU:

```text
lab.home.arpa
├── Domain Controllers
│   └── DC01
├── Lab-Computers
│   └── CLIENT01
└── Lab-Users
    ├── IT
    │   ├── testuser
    │   └── GG_IT_Users
    ├── HR
    │   ├── hruser
    │   ├── GG_HR_Users
    │   └── DL_HR_Files_RW
    └── Sales
        └── GG_Sales_Users
```

![Departmental OUs in AD Users and Computers](../assets/screenshots/fig-07-lab02-01-ad-organizational-units.png)

The group names reflect their roles: `GG_` for a **global** security group (who belongs to a department), and `DL_` for a **domain-local** security group (who gets access to a resource).

For the HR share, the chain is:

```text
hruser → GG_HR_Users → DL_HR_Files_RW → HR-Files permissions
```

This follows **AGDLP** (Accounts → Global → Domain Local → Permissions). Adding another HR user only requires changing the global group membership; the file ACL stays the same.

![HR user and nested security group membership](../assets/screenshots/fig-08-lab02-08-agdlp-nested-security-groups.png)

## A Group Policy that actually changes something

I created `GPO_HR_Restrict_ControlPanel`, linked it to the **HR OU**, and enabled this user setting:

`User Configuration → Policies → Administrative Templates → Control Panel → Prohibit access to Control Panel and PC settings`

![HR Control Panel restriction enabled](../assets/screenshots/fig-09-lab02-03-gpo-control-panel-restriction-enabled.png)

Then I signed into CLIENT01 with `LAB\hruser` and ran:

```powershell
gpupdate /force
gpresult /r /scope user
control.exe
```

`gpresult` listed the HR-specific GPO as applied, and Windows blocked Control Panel with its normal administrative restriction message.

![gpresult shows the HR policy](../assets/screenshots/fig-10-lab02-05-hr-gpresult-applied-gpo.png)

![Control Panel blocked for HR](../assets/screenshots/fig-11-lab02-06-hr-control-panel-blocked.png)

I signed out and tested `LAB\testuser` on **the same CLIENT01 VM**. Control Panel opened normally. The restriction followed the HR account's OU. I didn't have to change any local setting on CLIENT01 between logins.

![Control Panel allowed for the IT account](../assets/screenshots/fig-12-lab02-07-it-control-panel-allowed.png)

## HR file share using AGDLP

On DC01, I created `C:\LabShares\HR-Files` and shared it as `\\DC01\HR-Files`.

| Permission layer | Entries |
| --- | --- |
| **NTFS** on `HR-Files` | `SYSTEM` and `Administrators`: Full Control; `DL_HR_Files_RW`: Modify |
| **SMB share** | `Administrators`: Full Control; `DL_HR_Files_RW`: Change + Read |
| Inheritance | Disabled on `HR-Files` |
| Broad share access | Removed `Everyone` from the share ACL |

Windows checks both SMB share permissions and NTFS permissions for access over the network. I assigned permissions to the domain-local group, so I can manage HR membership without editing the folder ACL.

One mistake I caught during setup: I initially put the HR ACL on the parent `C:\LabShares` folder. That would have been the wrong place once other department folders were created. I removed it from the parent and applied it to `HR-Files` itself.

![NTFS permissions on HR-Files](../assets/screenshots/fig-13-lab02-09-hr-files-ntfs-permissions.png)

![SMB permissions for the HR group](../assets/screenshots/fig-14-lab02-10-hr-files-smb-share-permissions.png)

## Testing both outcomes

From CLIENT01, signed in as `LAB\hruser`, I opened `\\DC01\HR-Files` and created a text file.

![HR can create a file on the share](../assets/screenshots/fig-15-lab02-12-hr-file-creation-success.png)

Then I switched to `LAB\testuser` and tried the same network path. Windows returned **You do not have permission to access this share**.

![IT denied access to the HR folder](../assets/screenshots/fig-16-lab02-13-it-access-to-hr-share-denied.png)

I allowed access for the HR resource group and administrators. Since the IT account wasn't included, its access attempt failed without an explicit deny rule.

## Notes

- OUs are useful for **organizing accounts and targeting GPOs**. They're not file-access permission groups.
- Group membership and resource permissions are easier to maintain separately using AGDLP.
- For SMB access, share permissions and NTFS ACLs both matter.
- I tested access with both the HR and IT accounts. The IT test was useful for catching overly broad permissions.

**Result:** The HR GPO applied to HR users. HR could create files on the share, and IT was denied access.

**Next:** [Lab 03 — VLANs and physical networking](03-vlans-and-routing.md)
