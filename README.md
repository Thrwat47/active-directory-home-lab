# Active Directory Home Lab (VMware)

**Built:** October 2025



**Link to updated repo:** > 🔄 **Updated version available:** [NEW-REPO-NAME](https://github.com/Thrwat47/active-directory-home-lab/blob/main/docs/lab-update-v1.md)

## Overview
A home lab built on VMware Workstation to practice Windows Server administration and Active Directory. It includes a forest with a root domain and child domains, a second forest connected by a forest trust, multiple domain controllers in two AD sites, an OU structure, DHCP failover, Group Policy tested on a domain-joined Windows 11 client, and a Rocky Linux machine joined to the domain.

## Lab Environment
- **Hypervisor:** VMware Workstation
- **Main DC:** Windows Server 2022 Datacenter (Evaluation), hostname `DC`, domain `tacademy.local`
- **Other VMs:** additional domain controllers (`ADC`, `CORE2`, `CHILD`), the domain controller of the second forest (`DC-TREE`), a Windows 11 client, Rocky Linux, and Kali Linux
- **Main subnet:** `10.11.100.0/24` (LAN1), plus a second DHCP scope on LAN2 (`192.168.100.0/24`)

## Domain Structure

**Forest 1: `tacademy.local`**
- **Root domain:** `tacademy.local`
- **Child domains:** `core.tacademy.local` and `training.tacademy.local`
- **Domain controllers (all Global Catalog) in `tacademy.local`:** `DC` and `ADC` (Cairo site), and `CORE2` (Alex site)

**Forest 2: `macademy.local`**
- A separate forest with its own root domain and domain controller (`DC-TREE`), connected to `tacademy.local` with a **forest trust**

## Organizational Units
Under the `tacademy` OU I created departments, each with its own `Users` and `Computers` sub-OUs:
- HR
- IT
- Sales
- Guest
- Admins

I also created security groups (for example, an `HR` group), joined a Windows 11 client and a Rocky Linux machine to the domain, and applied Group Policy per OU.

## Active Directory Sites and Services
- Two sites: **Cairo** and **Alex**
- Subnet `10.11.100.0/24` assigned to the Cairo site
- Domain controllers placed in sites: `ADC`, `CHILD`, and `DC` in Cairo, and `CORE2` in Alex. The domain controller of the second forest (`DC-TREE`) is also placed in the Cairo site.

## DHCP
- DHCP server role installed on the DC and authorized in Active Directory
- **Two scopes:**
  - LAN1 (`10.11.100.0`): address pool, one excluded address, a reservation for a client, and scope options (router, DNS server, DNS domain name, NTP)
  - LAN2 (`192.168.100.0`): address pool with excluded ranges, plus router, DNS, and NTP options
- **DHCP failover** configured in **load balance mode** (50/50) with the partner server `ADC`
- DNS dynamic updates enabled, with A and PTR records discarded when a lease is deleted

## Group Policy (GPO)
I created and tested several Group Policy Objects using Group Policy Management on the `tacademy.local` domain:

| GPO | What it does |
|-----|--------------|
| **Background** | Sets a custom desktop wallpaper from a network share and prevents users from changing the desktop background |
| **HideAllDrives** | Hides all drives in My Computer, restricts access to the D drive, and removes Properties from the Computer icon context menu |
| **DisableRun** | Removes the Run command from the Start Menu |
| **RemoveClock** | Removes the clock from the system notification area (linked to the `Guest` and `IT/Users` OUs) |
| **HR-Policy-ControlPanel** | Blocks access to Control Panel and PC settings, and denies read/write/execute access to CD and DVD drives |
| **MapNetworkDrive** | Maps a shared folder as drive `Z:` for users using Group Policy Preferences |
| **DenyCD** (Starter GPO) | A reusable template that denies CD/DVD access |

I also worked with security filtering and delegation on the GPOs, and linked GPOs to specific OUs.

### GPO Testing
I joined a Windows 11 client to the domain, signed in with a user from the `tacademy/Guest/Users` OU, and verified the policies with `gpresult /r`. The `RemoveClock` and `HideAllDrives` GPOs were listed under **Applied Group Policy Objects**, which confirmed that the policies were delivered from the domain controller and applied to the user.

## Linux Integration (Rocky Linux)
I joined a Rocky Linux machine to the `tacademy.local` domain using `realmd` and `SSSD`:
1. Pointed the machine's DNS to the domain controller and set a fully qualified hostname.
2. Installed the required packages (`realmd`, `sssd`, `oddjob`, `adcli`, `krb5-workstation`).
3. Checked that the domain was visible with `realm discover`.
4. Joined the domain with `realm join`.
5. Verified the join with `realm list` and `id user@tacademy.local`.

The machine then appeared as a computer object in Active Directory, which showed how Linux can be managed in a mixed Windows and Linux environment.

## Other Tools Used
- **ADSI Edit** to view the Configuration, Schema, and naming contexts
- **Active Directory Domains and Trusts** to manage the domains and the forest trust
- **gpupdate** and **gpresult** to apply and verify Group Policy
- **realmd** and **SSSD** to integrate Linux with Active Directory

## What I Learned
- How to build an Active Directory forest and how child domains relate to the root domain.
- The difference between a child domain (shares the parent's namespace and forest) and a separate forest connected with a trust, and how a forest trust lets users in one forest access resources in another.
- How to organize users, computers, and groups with OUs.
- How AD sites and subnets control which domain controllers clients use.
- How DHCP scopes, reservations, and options work, and how failover load balancing shares a scope between two servers.
- How DNS and DHCP work together in a domain, and why a client needs the right DNS server to find the domain.
- How Group Policy controls user and computer settings, and how linking a GPO to an OU (and inheritance) decides who gets it.
- The difference between Administrative Templates (policies) and Group Policy Preferences (like drive mapping), and how Starter GPOs work as templates.
- How to verify Group Policy with `gpupdate /force` and `gpresult /r`.
- How to join a Linux machine to Active Directory with `realm join` and SSSD.
- This lab gave me hands-on practice for the kind of environment I will monitor as a SOC analyst.

## Screenshots
<img width="1920" height="1040" alt="Screenshot 2026-10-15 075610" src="https://github.com/user-attachments/assets/3f1eeb21-c1e4-471a-ba1e-a1c52cc54edf" /><img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/ff8e3f89-ec82-49c1-8d8d-3baf0c88b83e" /><img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/aa9cce07-a43c-42e3-b0b4-cc047881163a" />
<img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/2e049a7b-34a9-49a8-a8f4-472fe7db5509" /><img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/c155d2f5-c5b0-45f4-ac10-7228ae53973b" /><img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/7d13b23f-6c2c-413a-91bb-18ef596c11ad" /><img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/14bd4922-4822-4d21-a05c-5252f7727f94" />






