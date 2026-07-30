# Azure Active Directory Home Lab

A self-directed lab where I deployed and configured a functional Active Directory environment on Azure-hosted Windows Server infrastructure, covering domain services, organizational unit design, identity/access management, and troubleshooting of realistic connectivity and authentication issues.

## Project Overview

I built a two-VM Active Directory environment from scratch in Microsoft Azure:

- **DC01**: Windows Server 2022, promoted to a Domain Controller for a new forest (`lab.local`)
- **CLIENT01**: Windows Server 2022 configured and joined as a domain client

The goal was to practice the identity and access management responsibilities that come up daily in Help Desk and Systems Administrator roles: creating and organizing user accounts, managing group-based access, and diagnosing real connectivity failures rather than just following a checklist.

## Business Scenario

Active Directory is still the backbone of identity and access management for most mid-size and enterprise companies, and even organizations that have adopted cloud services typically run AD alongside cloud identity (like Microsoft Entra ID) in a hybrid model. A new IT hire is expected to understand how user accounts, groups, and permissions work together, and how to troubleshoot when authentication or connectivity breaks.

This lab simulates that environment: a small multi-branch company with a branch office in Oakbrook, IL needs its users organized, grouped by role, and provisioned with least-privilege access, which is the same task a Help Desk tech would be assigned in their first weeks on the job.

## Skills Demonstrated

- Active Directory Domain Services (AD DS) installation and domain controller promotion
- Organizational Unit (OU) design based on real-world delegation/policy boundaries
- User account creation and management
- Security group creation and role-based group membership (least privilege)
- Domain-joining a client machine and troubleshooting DNS dependency issues
- Delegated administrative permissions (password reset rights for a Helpdesk group)
- Domain-wide password policy configuration
- Azure VM provisioning, networking, and quota troubleshooting
- Out-of-band remote access (Azure Serial Console) for recovering a misconfigured VM
- Cost-conscious cloud hygiene (deallocating and deleting resources after use)

## Architecture

```
Azure Resource Group: ad-lab-rg
│
├── DC01 (Windows Server 2022, Domain Controller)
│   ├── Forest/Domain: lab.local
│   ├── NetBIOS: LAB
│   ├── AD DS + DNS + Global Catalog
│   └── Static private IP: 172.16.0.4
│
├── CLIENT01 (Windows Server 2022, domain-joined client)
│   └── Static private IP: 172.16.0.5
│
└── Shared Virtual Network (same VNet/subnet for both VMs)

Active Directory Structure (lab.local)
│
├── _Branches
│   └── Oakbrook
│       ├── Users        (3 domain user accounts)
│       ├── Workstations (CLIENT01 computer object)
│       └── Laptops
│
└── _Groups
    ├── Helpdesk    (delegated password-reset rights on Oakbrook/Users)
    ├── Accounting
    └── ITSupport
```

**Design decisions:**
- **Branch-based OUs** (rather than flat `_Users`/`_Computers` containers) reflect how real multi-site organizations structure Group Policy and delegation boundaries.
- **Centralized `_Groups` OU** at the domain root, separate from branch OUs, since groups represent roles/departments rather than locations and don't belong nested under a specific branch.
- **Global security groups** used to grant access, rather than assigning permissions directly to user accounts, which is the least-privilege, scalable pattern used in production AD environments.

## Implementation Summary

**1. Domain controller promotion**
Provisioned `DC01` in Azure and promoted it to a domain controller, creating a new forest (`lab.local`).

<img width="600" alt="VM creation in Azure" src="https://github.com/user-attachments/assets/9481a32d-f7fb-45ac-a081-ccfef2da6e54" />
<img width="500" alt="Domain promotion confirmed via whoami" src="https://github.com/user-attachments/assets/16e91cb6-2bc5-431d-a0e4-27e6b31b44d8" />

**2. Organizational Unit structure**
Built a branch-based OU structure (`_Branches → Oakbrook → Users / Workstations / Laptops`).

<img width="500" alt="OU structure in Active Directory Users and Computers" src="https://github.com/user-attachments/assets/6e44a0e7-137a-4805-b8dd-ae5153efeeb4" />

**3. User accounts**
Created domain user accounts and placed them in the correct branch OU.

<img width="450" alt="Domain user accounts created" src="https://github.com/user-attachments/assets/25558975-e1f8-4256-b091-dfd0fba0c4af" />

**4. Security groups**
Created a centralized `_Groups` OU with three role-based security groups (Helpdesk, Accounting, ITSupport) and assigned users accordingly.

<img width="500" alt="Security groups created" src="https://github.com/user-attachments/assets/21bede04-8e6c-4767-82bb-93edc125feaa" />

**5. Client domain join**
Provisioned `CLIENT01`, configured it to use `DC01` for DNS, and joined it to the domain (see Troubleshooting Log for issues encountered during this step).

<img width="500" alt="CLIENT01 network configuration troubleshooting" src="https://github.com/user-attachments/assets/0f4650e4-0b99-4796-8806-a7ec640bece8" />

**6. Authentication test**
Logged in as a domain user on `CLIENT01` and confirmed correct group membership via `whoami /groups`.

<img width="600" alt="whoami groups output confirming domain authentication" src="https://github.com/user-attachments/assets/d9f398e9-df75-4554-b809-8454f5a321a5" />

**7. Bonus administrative tasks**
Delegated password-reset rights to the Helpdesk group, configured a domain-wide password policy, and moved the client computer object into its correct OU.

<img width="450" alt="Delegated password reset permissions" src="https://github.com/user-attachments/assets/f5eb5d88-82cd-4fc9-8af6-d9b831f9f8c1" />
<img width="500" alt="Domain password policy configuration" src="https://github.com/user-attachments/assets/f30de49b-c8e5-4eab-b5b1-e13eb482be38" />
<img width="450" alt="CLIENT01 moved into correct branch OU" src="https://github.com/user-attachments/assets/b5b8a419-3f12-49f6-9cae-9566f3205669" />

## Troubleshooting Log

Real issues encountered and resolved during the build, not simulated for this writeup.

### Issue 1: CLIENT01 lost RDP access after manual network configuration

- **Symptom:** After manually configuring IPv4 settings on CLIENT01, RDP connections failed with error 0x204.
- **Investigation:** Reviewed the network configuration that had just been applied.
- **Root cause:** The static IP configuration accidentally duplicated DC01's IP address (172.16.0.4) on CLIENT01, and the subnet mask/default gateway were incomplete, which knocked CLIENT01 off the network entirely.
- **Fix:** Used Azure Serial Console, an out-of-band access method that works even when standard network connectivity (and therefore RDP) is down, to correct the static IP (172.16.0.5), subnet mask (255.255.255.0), and default gateway (172.16.0.1).
- **Result:** RDP access restored, confirmed connectivity.

### Issue 2: Domain join failed due to DNS resolution

- **Symptom:** Attempting to join CLIENT01 to `lab.local` returned: *"An Active Directory Domain Controller for the domain lab.local could not be contacted."*
- **Investigation:** Ran `nslookup lab.local` on CLIENT01. The DNS server in use was an IPv6 placeholder address, returning no response.
- **Root cause:** IPv6 was taking priority over the manually configured IPv4 DNS setting, so CLIENT01 wasn't actually querying DC01 for name resolution.
- **Fix:** Disabled IPv6 on CLIENT01's network adapter, forcing it to use the correct IPv4 DNS server (DC01, 172.16.0.4).
- **Verification:** Re-ran `nslookup lab.local` and confirmed it resolved correctly to DC01.
- **Result:** Domain join completed successfully.

### Issue 3: Azure VM size quota exceeded

- **Symptom:** Initial VM deployment failed with `QuotaExceeded` on the `Basv2` VM family.
- **Investigation:** Checked the Azure "Usage + Quotas" blade and found the subscription had a default quota of 0 for all B-series VM families in the target region.
- **Root cause:** New/free-tier Azure subscriptions don't automatically include quota for every VM family, and B-series (the cheapest general-purpose option) wasn't pre-approved.
- **Fix:** Identified an available VM family with approved quota (Ev5 series) and redeployed using that size instead.
- **Result:** Deployment succeeded. Noted for future labs: check quota availability before assuming a specific "tutorial" VM size will be deployable.

## Lessons Learned

- Active Directory issues are rarely caused by AD itself. DNS and networking are almost always the actual root cause, which matched what I'd read but hadn't experienced firsthand until CLIENT01's domain join failed.
- Manual network configuration carries real risk, as seen with the duplicate-IP incident, since small mistakes can fully disconnect a machine. Recovering from that taught me the value of out-of-band access methods like Serial Console.
- Cloud environments have platform-level constraints (quotas, region availability) that don't show up in a tutorial written for a different subscription type, so real-world troubleshooting can start before you even touch the technology you're trying to learn.
- Group-based access control, rather than direct permission assignment, is a foundational AD concept that scales. I understand not just how to do it, but why it's the standard.

## Interview Explanation

*"I built a two-VM Active Directory environment in Azure, a domain controller and a domain-joined client, to practice the identity and access management work that comes up in Help Desk and Sysadmin roles. I designed a branch-based OU structure, created users and role-based security groups following least-privilege principles, and delegated limited admin rights to a Helpdesk group. Along the way, I hit and resolved two real issues: a network misconfiguration that knocked my client VM offline, which I fixed using Azure's Serial Console since normal RDP access was down, and a DNS resolution problem that was blocking the domain join, which I diagnosed using `nslookup` and traced to an IPv6 priority conflict. Both taught me that most AD problems trace back to networking and DNS rather than AD itself, which is exactly the kind of troubleshooting mindset I'd bring to a Help Desk role."*

## Resume Bullet Draft *(pending your approval, not final until you sign off)*

> Deployed and configured a two-VM Active Directory environment in Microsoft Azure, including domain controller promotion, branch-based OU design, and role-based security group access; independently diagnosed and resolved DNS and network misconfiguration issues using `nslookup` and Azure Serial Console.

---

*Lab based on a self-guided Active Directory training exercise. All infrastructure was provisioned in Microsoft Azure and deprovisioned after completion to avoid ongoing cost.*
