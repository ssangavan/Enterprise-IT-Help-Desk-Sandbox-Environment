# Enterprise IT Help Desk Sandbox Environment

A virtualized corporate IT environment built to practice the day-to-day work of a Tier-1/Tier-2 help desk and junior systems administrator: standing up a domain, managing identities, enforcing security policy, controlling file access, and patching endpoints.

**Completed:** June 2026
**Tools:** Oracle VirtualBox · Windows Server 2022 · Windows 11 Enterprise · Active Directory Domain Services · Group Policy · NTFS Permissions · Action1 RMM

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Lab Architecture](#2-lab-architecture)
3. [Phase 1 - Infrastructure Deployment](#3-phase-1---infrastructure-deployment)
4. [Phase 2 - Identity & Access Management](#4-phase-2---identity--access-management-iam)
5. [Phase 3 - Security & Policy Enforcement (GPO)](#5-phase-3---security--policy-enforcement-gpo)
6. [Phase 4 - Data Security & File Systems](#6-phase-4---data-security--file-systems)
7. [Phase 5 - Endpoint & Patch Management](#7-phase-5---endpoint--patch-management)
8. [Troubleshooting & Lessons Learned](#8-troubleshooting--lessons-learned)
9. [Skills Demonstrated](#9-skills-demonstrated)
10. [Future Improvements](#10-future-improvements)

---

## 1. Project Overview

**Goal:** Simulate a small corporate network so I could practice real help desk workflows (onboarding, password resets, access requests, patching) in a safe environment.

**What I built:**

| Area | What I did |
|---|---|
| Infrastructure | Deployed a Windows Server 2022 Domain Controller and a Windows 11 Enterprise client in VirtualBox, joined to a new sangavantech.com |
| IAM | Created OUs, users, and security groups in AD DS; simulated onboarding and Tier-1 tickets |
| Security | Enforced password, lockout, and desktop restriction policies through GPOs |
| Data security | Built a central file share with NTFS permissions mapped to AD security groups |
| Endpoint mgmt | Used Action1 RMM to audit vulnerabilities, generate compliance reports, and deploy patches/software |

---

## 2. Lab Architecture

### Virtual Machines

| VM Name | OS | Role | Hostname | IP Address |
|---|---|---|---|---|
| DC01 | Windows Server 2022 | Domain Controller, DNS, File Server | `[TO-DC01]` | `[10.1.10.1]` (static) |
| CLIENT01 | Windows 11 Enterprise | Domain-joined endpoint | `[DESKTOP01]` | `[10.1.10.3]` |

**Domain name:** `[sangavantech.com]`
**Network mode:** VirtualBox Host-Only Adapter for the isolated lab network (static 10.1.10.0/24 addresses). 
A Bridged Adapter was enabled temporarily on the VMs whenever internet access was required (Action1 agent enrollment).

### Network Diagram

                +-----------------------------+
                |   Host Machine (VirtualBox) |
                +--------------+--------------+
                               |
                   [Host-Only Network 10.1.10.0/24]
                   (+ temporary Bridged Adapter for internet)
                               |
          +--------------------+--------------------+
          |                                         |
 +--------+---------+                     +---------+--------+
 |  TO-DC01         |                     |  DESKTOP01       |
 |  10.1.10.1       |<--- domain join --->|  10.1.10.3       |
 |  Server 2022     |                     |  Windows 11      |
 |  AD DS, DNS,     |                     |  Action1 agent   |
 |  File Shares     |                     |                  |
 +------------------+                     +------------------+
                \                              /
                 \------ Action1 (cloud) ------/

> 📸 **Screenshot:** `screenshots/00-overview/network-diagram.png`
> *(Optional: replace the ASCII diagram above with a draw.io / diagrams.net export.)*

> 📸 **Screenshot:** `screenshots/00-overview/virtualbox-manager.png`
> *VirtualBox Manager showing both VMs.*

---

## 3. Phase 1 - Infrastructure Deployment

### 3.1 Create the Windows Server 2022 VM

1. Created a new VM in VirtualBox with `[X]` GB RAM, `[X]` CPU cores, and `[X]` GB disk.
2. Mounted the Windows Server 2022 ISO and completed installation (Desktop Experience).
3. Installed VirtualBox Guest Additions.

> 📸 `screenshots/01-infrastructure/01-dc-vm-settings.png` - *VirtualBox settings for the DC VM (System, Network tabs).*
> 📸 `screenshots/01-infrastructure/02-server-install-complete.png` - *Server Manager dashboard after first login.*

### 3.2 Configure Server Basics

Set the hostname and a static IP address.

```powershell
Rename-Computer -NewName "[DC-HOSTNAME]" -Restart

New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "[STATIC-IP]" `
  -PrefixLength 24 -DefaultGateway "[GATEWAY-IP]"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "127.0.0.1"
```

> 📸 `screenshots/01-infrastructure/03-static-ip-config.png` - *`ipconfig /all` output on the DC.*

### 3.3 Install AD DS and Promote to Domain Controller

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "[yourdomain.local]" `
  -DomainNetbiosName "[YOURDOMAIN]" `
  -InstallDns -Force
```

> 📸 `screenshots/01-infrastructure/04-adds-role-install.png` - *Add Roles and Features wizard / install progress.*
> 📸 `screenshots/01-infrastructure/05-dc-promotion.png` - *Promotion to domain controller (new forest).*
> 📸 `screenshots/01-infrastructure/06-aduc-console.png` - *Active Directory Users and Computers showing the new domain.*

### 3.4 Deploy the Windows 11 Client and Join the Domain

1. Created the Windows 11 Enterprise VM (`[X]` GB RAM, `[X]` GB disk) on the same virtual network.
2. Pointed the client's DNS at the DC (`[STATIC-IP]`).
3. Joined the machine to the domain.

```powershell
Add-Computer -DomainName "[yourdomain.local]" -Credential "[YOURDOMAIN]\Administrator" -Restart
```

> 📸 `screenshots/01-infrastructure/07-client-dns-settings.png` - *Client network adapter DNS pointing to the DC.*
> 📸 `screenshots/01-infrastructure/08-client-domain-join.png` - *System Properties showing the domain membership.*
> 📸 `screenshots/01-infrastructure/09-client-computer-in-ad.png` - *Client computer object visible in ADUC.*

---

## 4. Phase 2 - Identity & Access Management (IAM)

### 4.1 Organizational Unit (OU) Structure

```
[yourdomain.local]
├── Corp
│   ├── Users
│   │   ├── IT
│   │   ├── HR
│   │   └── Finance
│   ├── Computers
│   └── Groups
```

> *Adjust the tree above to match what you actually built.*

```powershell
New-ADOrganizationalUnit -Name "Corp" -Path "DC=[yourdomain],DC=[local]"
New-ADOrganizationalUnit -Name "Users" -Path "OU=Corp,DC=[yourdomain],DC=[local]"
# ...repeat for each OU
```

> 📸 `screenshots/02-iam/01-ou-structure.png` - *ADUC expanded to show the full OU tree.*

### 4.2 Security Groups

| Group | Type | Purpose |
|---|---|---|
| `[GG-IT]` | Global Security | IT staff |
| `[GG-HR]` | Global Security | HR staff |
| `[GG-Finance]` | Global Security | Finance staff |

```powershell
New-ADGroup -Name "[GG-HR]" -GroupScope Global -GroupCategory Security `
  -Path "OU=Groups,OU=Corp,DC=[yourdomain],DC=[local]"
```

> 📸 `screenshots/02-iam/02-security-groups.png` - *Groups OU with all security groups listed.*

### 4.3 User Onboarding (Simulated)

Created user accounts and assigned them to the right OU and groups, as if processing a new-hire ticket.

```powershell
New-ADUser -Name "[First Last]" -SamAccountName "[flast]" `
  -UserPrincipalName "[flast]@[yourdomain.local]" `
  -Path "OU=HR,OU=Users,OU=Corp,DC=[yourdomain],DC=[local]" `
  -AccountPassword (Read-Host -AsSecureString "Temp password") `
  -ChangePasswordAtLogon $true -Enabled $true

Add-ADGroupMember -Identity "[GG-HR]" -Members "[flast]"
```

> 📸 `screenshots/02-iam/03-new-user-wizard.png` - *Creating a new user in ADUC.*
> 📸 `screenshots/02-iam/04-user-group-membership.png` - *User's "Member Of" tab.*
> 📸 `screenshots/02-iam/05-user-first-login.png` - *Forced password change at first logon on the client.*

### 4.4 Tier-1 Ticket Simulations

| Ticket | Scenario | Resolution |
|---|---|---|
| #001 | User forgot password | Reset password in ADUC, required change at next logon |
| #002 | Account locked out | Unlocked account, checked lockout source |
| #003 | New hire onboarding | Created account, added to department group |
| #004 | Employee departure | Disabled account, moved to Disabled Users OU, removed from groups |
| `[#005]` | `[Add your own]` | `[Resolution]` |

> 📸 `screenshots/02-iam/06-password-reset.png`
> 📸 `screenshots/02-iam/07-account-unlock.png`
> 📸 `screenshots/02-iam/08-offboarding-disabled-account.png`

---

## 5. Phase 3 - Security & Policy Enforcement (GPO)

All GPOs were created in **Group Policy Management**, linked to the appropriate OU, and tested from the Windows 11 client.

### 5.1 Password & Account Lockout Policy

| Setting | Value |
|---|---|
| Minimum password length | `[e.g., 12]` characters |
| Password complexity | Enabled |
| Maximum password age | `[e.g., 90]` days |
| Password history | `[e.g., 24]` passwords |
| Account lockout threshold | `[e.g., 5]` invalid attempts |
| Lockout duration | `[e.g., 30]` minutes |
| Reset lockout counter after | `[e.g., 30]` minutes |

> 📸 `screenshots/03-gpo/01-gpmc-overview.png` - *GPMC showing GPOs and their OU links.*
> 📸 `screenshots/03-gpo/02-password-policy-settings.png` - *Password policy configuration in the GPO editor.*
> 📸 `screenshots/03-gpo/03-lockout-policy-settings.png` - *Account lockout policy configuration.*

**Testing:**
- Attempted a weak password reset and confirmed it was rejected.
- Entered the wrong password `[X]` times and confirmed the account locked.

> 📸 `screenshots/03-gpo/04-weak-password-rejected.png`
> 📸 `screenshots/03-gpo/05-account-locked-message.png`

### 5.2 Local Desktop Restrictions

| Restriction | GPO Path |
|---|---|
| `[e.g., Block Control Panel access]` | User Config > Admin Templates > Control Panel |
| `[e.g., Disable Command Prompt]` | User Config > Admin Templates > System |
| `[e.g., Prevent changing wallpaper]` | User Config > Admin Templates > Control Panel > Personalization |

> 📸 `screenshots/03-gpo/06-desktop-restriction-gpo.png` - *GPO editor with restriction settings.*
> 📸 `screenshots/03-gpo/07-restriction-in-action.png` - *Client showing the restriction blocking the action.*

### 5.3 Verifying Policy Application

```cmd
gpupdate /force
gpresult /r
gpresult /h C:\gpreport.html
```

> 📸 `screenshots/03-gpo/08-gpresult-output.png` - *`gpresult /r` on the client showing applied GPOs.*

---

## 6. Phase 4 - Data Security & File Systems

### 6.1 Folder Structure

```
D:\CompanyShare\          (or C:\CompanyShare\)
├── HR
├── Finance
├── IT
└── Public
```

```powershell
New-Item -Path "[D:\CompanyShare]" -ItemType Directory
"HR","Finance","IT","Public" | ForEach-Object {
  New-Item -Path "[D:\CompanyShare]\$_" -ItemType Directory
}
```

> 📸 `screenshots/04-file-share/01-folder-structure.png`

### 6.2 Share the Root Folder

Shared `[CompanyShare]` with Share permissions set to `[Authenticated Users / Everyone: Change]`, and let **NTFS permissions** do the real access control (the least-restrictive-share / most-restrictive-NTFS model).

> 📸 `screenshots/04-file-share/02-share-permissions.png` - *Advanced Sharing > Permissions.*

### 6.3 NTFS Permissions Mapped to AD Security Groups

Inheritance was disabled on each department folder, and access was granted only to the matching group.

| Folder | `[GG-IT]` | `[GG-HR]` | `[GG-Finance]` | Domain Users |
|---|---|---|---|---|
| HR | Modify | Modify | - | - |
| Finance | Modify | - | Modify | - |
| IT | Full Control | - | - | - |
| Public | Modify | Read | Read | Read |

> *Edit this matrix so it matches your actual configuration.*

```powershell
icacls "[D:\CompanyShare]\HR" /inheritance:r
icacls "[D:\CompanyShare]\HR" /grant "[YOURDOMAIN]\[GG-HR]:(OI)(CI)M"
icacls "[D:\CompanyShare]\HR" /grant "[YOURDOMAIN]\[GG-IT]:(OI)(CI)M"
```

> 📸 `screenshots/04-file-share/03-ntfs-security-tab-hr.png` - *Security tab of the HR folder.*
> 📸 `screenshots/04-file-share/04-ntfs-security-tab-finance.png`
> 📸 `screenshots/04-file-share/05-inheritance-disabled.png` - *Advanced Security Settings showing inheritance removed.*

### 6.4 Access Testing from the Client

Logged into the client as different users to confirm access works as designed.

| User | Group | Tried to access | Expected | Result |
|---|---|---|---|---|
| `[hr_user]` | HR | `\\[DC]\CompanyShare\HR` | Allowed | ✅ |
| `[hr_user]` | HR | `\\[DC]\CompanyShare\Finance` | Denied | ✅ |
| `[fin_user]` | Finance | `\\[DC]\CompanyShare\Finance` | Allowed | ✅ |
| `[fin_user]` | Finance | `\\[DC]\CompanyShare\HR` | Denied | ✅ |

> 📸 `screenshots/04-file-share/06-access-allowed.png`
> 📸 `screenshots/04-file-share/07-access-denied.png`

*(Optional)* Mapped the share as a network drive via GPO Drive Maps:

> 📸 `screenshots/04-file-share/08-mapped-drive-gpo.png`

---

## 7. Phase 5 - Endpoint & Patch Management

### 7.1 Action1 Setup and Agent Deployment

1. Created an Action1 account and organization.
2. Downloaded the agent installer from the Action1 console.
3. Installed the agent on `[DC01 and/or CLIENT01]`.
4. Confirmed endpoints were reporting in.

> 📸 `screenshots/05-action1/01-action1-dashboard.png` - *Action1 dashboard.*
> 📸 `screenshots/05-action1/02-agent-install.png` - *Agent installation on the endpoint.*
> 📸 `screenshots/05-action1/03-endpoints-online.png` - *Endpoints list showing online status.*

### 7.2 Vulnerability Audit

Reviewed missing updates and known vulnerabilities per endpoint, sorted by severity.

> 📸 `screenshots/05-action1/04-vulnerability-overview.png`
> 📸 `screenshots/05-action1/05-missing-updates-detail.png`

**Findings:**
- `[X]` missing OS updates identified on `[endpoint]`
- `[X]` third-party applications flagged as outdated
- `[X]` critical/high severity vulnerabilities

### 7.3 Compliance Reporting

Generated an endpoint compliance report.

> 📸 `screenshots/05-action1/06-compliance-report.png`
> *(Optional: export the report as PDF into `/reports` after redacting any sensitive details.)*

### 7.4 Automated Patch & Software Deployment

- Created an automation/policy to deploy critical OS patches to `[endpoint group]`.
- Deployed third-party software `[e.g., 7-Zip, Chrome, Notepad++]` remotely.

> 📸 `screenshots/05-action1/07-patch-automation-policy.png`
> 📸 `screenshots/05-action1/08-patch-deployment-status.png`
> 📸 `screenshots/05-action1/09-software-deployment.png`
> 📸 `screenshots/05-action1/10-post-patch-compliance.png` - *Dashboard after patching, showing improved compliance.*

---

## 8. Troubleshooting & Lessons Learned

| Problem | Cause | Fix |
|---|---|---|
| Client couldn't find the domain | Client DNS pointed to the router/ISP instead of the DC | Set the client's DNS to the DC's IP |
| `[Your issue]` | `[Cause]` | `[Fix]` |
| `[Your issue]` | `[Cause]` | `[Fix]` |

**Key takeaways:**
- DNS is the foundation of Active Directory; most join/login problems trace back to it.
- Assigning NTFS permissions to groups rather than individual users makes access far easier to audit.
- `gpresult` and `gpupdate /force` are essential for verifying and troubleshooting GPOs.
- `[Add your own reflection]`

---

## 9. Skills Demonstrated

- Virtualization (Oracle VirtualBox)
- Windows Server 2022 administration
- Active Directory Domain Services (AD DS) and DNS
- User, group, and OU lifecycle management
- Group Policy design, linking, and troubleshooting
- NTFS and share permissions (least privilege)
- Remote Monitoring & Management (Action1)
- Patch and vulnerability management
- PowerShell automation
- Technical documentation

---

## 10. Future Improvements

- [ ] Deploy a help desk ticketing system (e.g., osTicket, GLPI, Spiceworks) and log the Tier-1 tickets formally
- [ ] Add a second DC for redundancy and replication
- [ ] Configure DHCP on the server
- [ ] Add Windows Server Update Services (WSUS) or compare with Action1
- [ ] Implement LAPS for local administrator passwords
- [ ] Script bulk user creation from CSV with PowerShell

---

## Repository Structure

```
it-helpdesk-lab/
├── README.md
├── scripts/
│   ├── 01-configure-dc.ps1
│   ├── 02-create-ous-groups.ps1
│   ├── 03-create-users.ps1
│   └── 04-create-shares-ntfs.ps1
├── screenshots/
│   ├── 00-overview/
│   ├── 01-infrastructure/
│   ├── 02-iam/
│   ├── 03-gpo/
│   ├── 04-file-share/
│   └── 05-action1/
└── reports/
    └── action1-compliance-report.pdf   (optional, redacted)
```

> **Note:** This is an isolated lab. All usernames, passwords, and IP addresses are for demonstration only. Passwords and any Action1 organization identifiers have been omitted or redacted.
