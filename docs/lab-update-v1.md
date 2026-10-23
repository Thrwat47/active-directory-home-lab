# tacademy.local: Segmented Active Directory Lab with Splunk SIEM




> 📌 **Earlier version:** [active-directory-home-lab](https://github.com/Thrwat47/active-directory-home-lab)


A home lab on VMware Workstation that simulates a small enterprise network: an Active Directory domain, five isolated network segments (one per OU), DHCP and DNS served by the domain controller, Group Policy, and a Splunk SIEM on Rocky Linux collecting the domain controller's Windows event logs.

The main focus is **network segmentation and least-privilege isolation** of a restricted "Guest" segment: what I built, how I tested it, the mistakes I found along the way, and how I diagnosed them (see [Troubleshooting log](#troubleshooting-log)).

---

## Tech stack

| Area | Tools |
|---|---|
| Virtualization | VMware Workstation (Host-only VMnets) |
| Directory | Windows Server 2022, Active Directory DS, AD Sites and Services |
| Network services | DHCP, DNS, IP Forwarding (routing), Windows Defender Firewall |
| Policy | Group Policy (GPMC) |
| Monitoring | Splunk Enterprise 10.4.1 on Rocky Linux, Splunk Universal Forwarder on the DC |
| Clients | Windows 11 domain-joined machines |

---

## Architecture

Each OU has its own network. The domain controller (`dc.tacademy.local`) is multi-homed: it has one NIC per network and acts as gateway, DHCP server and DNS server for each of them.

| OU | VLAN ID | VMware network | Subnet | DC address (gateway) | DHCP pool |
|---|---|---|---|---|---|
| Admins | 10 | VMnet10 | 10.10.10.0/24 | 10.10.10.1 | .50 - .250 |
| IT | 20 | VMnet11 | 10.10.20.0/24 | 10.10.20.1 | .50 - .250 |
| HR | 30 | VMnet12 | 10.10.30.0/24 | 10.10.30.1 | .50 - .250 |
| Sales | 40 | VMnet13 | 10.10.40.0/24 | 10.10.40.1 | .50 - .250 |
| Guest | 50 | VMnet14 | 10.10.50.0/24 | 10.10.50.1 | .50 - .250 |
| Original network | - | - | 10.11.100.0/24 | 10.11.100.100 | - |

```mermaid
flowchart LR
    subgraph DC["dc.tacademy.local (multi-homed, routes between networks)"]
        N10["VLAN10 10.10.10.1"]
        N20["VLAN20 10.10.20.1"]
        N30["VLAN30 10.10.30.1"]
        N40["VLAN40 10.10.40.1"]
        N50["VLAN50 10.10.50.1 (Forwarding OFF)"]
        N0["Ethernet0 10.11.100.100"]
    end
    A[Admins] --- N10
    I[IT] --- N20
    H[HR] --- N30
    S[Sales] --- N40
    G[Guest] --- N50
    O[ADC / CORE2 / Splunk] --- N0
```

> **Note:** The "VLANs" are separate VMware Host-only networks, not 802.1Q tagging. Using the domain controller as router, DHCP and DNS server at the same time is a lab simplification, not an enterprise design.

**Virtual networks (VMnet10 - VMnet14) and the DC's NICs:**

![VMware Virtual Network Editor](screenshots/01-vmware-virtual-networks.png)

![DC network interfaces and IP addresses](screenshots/02-dc-nics-and-ips.png)

---

## What is implemented

- **Five DHCP scopes** with gateway (003), DNS (006) and domain name (015) options per network.
- **Routing between networks** using IP Forwarding on the DC's NICs.
- **DNS cleanup for a multi-homed DC:** per-VLAN NICs do not register in DNS, and the extra A records for `dc` were removed so clients always resolve it to a reachable address.
- **AD Sites and Services subnets** for every network, so AD knows which network a machine comes from.
- **Domain-joined clients** verified with `nltest`, `gpupdate` and `gpresult`.
- **Group Policy** objects for restrictions (Run dialog, drives, clock, Control Panel, drive mapping, wallpaper, removable media).
- **Guest isolation (least privilege)**, described below.
- **Splunk SIEM** receiving the DC's Security, System and Application logs.

![DHCP scopes on the DC](screenshots/03-dhcp-scopes.png)

The Guest scope hands out the DC's Guest-side address (`10.10.50.1`) as gateway and DNS:

![Guest DHCP scope options](screenshots/04-dhcp-guest-scope-options.png)

![Guest DHCP lease](screenshots/05-dhcp-guest-lease.png)

---

## Guest network isolation

The Guest machine is **domain-joined**, so it is a restricted segment rather than a true guest network. It still needs the DC for logon and Group Policy, but nothing else.

**Goal (least privilege):** the Guest can reach only the DC services it needs, and cannot reach any other network.

### Two layers

| Layer | Controls | Mechanism |
|---|---|---|
| Routing | Traffic passing **through** the DC to other networks | IP Forwarding disabled on the Guest NIC |
| Host firewall | Traffic going **to** the DC itself | Windows Firewall inbound block rules scoped to `10.10.50.0/24` |

Disabling forwarding alone is not enough: it stops the Guest from reaching other VLANs, but the Guest can still connect to services running on the DC (SMB, LDAP, RDP, WinRM...).

### Layer 1: routing

```powershell
Set-NetIPInterface -InterfaceAlias "VLAN50" -Forwarding Disabled
```

### Layer 2: firewall rules on the DC

Windows Firewall block rules override allow rules, so the block rules exclude the ports the Guest legitimately needs instead of adding allow rules on top.

Ports left open to the Guest:

| Port | Purpose |
|---|---|
| TCP/UDP 53 | DNS |
| UDP 67 | DHCP |
| TCP/UDP 88 | Kerberos |
| UDP 123 | Time (W32Time) |
| TCP 135 and 49152-65535 | RPC |
| TCP/UDP 389 | LDAP |
| TCP 445 | SMB (SYSVOL / NETLOGON for Group Policy) |
| TCP/UDP 464 | Password change |
| TCP 3268 | Global Catalog |

```powershell
# Enable the firewall on all profiles and log dropped packets
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Allow
Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True -LogMaxSizeKilobytes 16384

# Block everything except the ports above, only for traffic coming from the Guest subnet
New-NetFirewallRule -DisplayName "GUEST-Block-TCP" -Direction Inbound -Action Block -Protocol TCP `
  -LocalPort "1-52","54-87","89-134","136-388","390-444","446-463","465-3267","3269-49151" `
  -RemoteAddress 10.10.50.0/24 -Profile Any

New-NetFirewallRule -DisplayName "GUEST-Block-UDP" -Direction Inbound -Action Block -Protocol UDP `
  -LocalPort "1-52","54-66","68-87","89-122","124-388","390-463","465-65535" `
  -RemoteAddress 10.10.50.0/24 -Profile Any
```

![GUEST-Block-TCP and GUEST-Block-UDP in Inbound Rules](screenshots/12-dc-firewall-inbound-block-rules.png)

### Verification

All tests run from the Guest machine (`10.10.50.50`, logged on as a domain user):

![Guest IP configuration and domain user](screenshots/06-guest-ipconfig-whoami.png)

| Test | Expected | Result |
|---|---|---|
| `Test-NetConnection 10.10.50.1 -Port 445` | Succeeds | Succeeded |
| `Test-NetConnection 10.10.50.1 -Port 389` | Succeeds | Succeeded |
| `Test-NetConnection 10.10.50.1 -Port 3389` | Fails | **Failed** (was open before the firewall was enabled) |
| `Test-NetConnection 10.10.50.1 -Port 5985` | Fails | Failed |
| `Test-NetConnection 10.10.20.50 -Port 445` (IT network) | Fails | Failed |
| `tracert 10.10.20.50` | Does not reach | Timed out on every hop |

Allowed ports (what the Guest needs for logon and Group Policy):

![Guest reaches SMB and LDAP on the DC](screenshots/07-guest-allowed-ports-445-389.png)

Blocked: WinRM on the DC, and the IT network:

![Guest blocked from WinRM and from the IT network](screenshots/08-guest-blocked-5985-and-it-445.png)

![tracert from Guest to the IT network times out](screenshots/09-guest-tracert-to-it.png)

**Before and after for RDP (port 3389) on the DC.** The first test succeeded because the Windows Firewall on the DC was off:

![Before: RDP on the DC reachable from Guest](screenshots/10-before-guest-rdp-3389-open.png)

![DC Windows Firewall off on all three profiles](screenshots/11-dc-firewall-off.png)

After enabling the firewall and rules, the same test fails, and the Guest still resolves the DC through DNS (`dc.tacademy.local` returns `10.11.100.100`):

![After: RDP blocked, DNS still works](screenshots/14-after-guest-rdp-3389-blocked.png)

**Proof from the DC itself.** The firewall log shows the dropped RDP attempt from the Guest (`DROP TCP 10.10.50.50 10.10.50.1 ... 3389`), plus dropped NetBIOS name-service traffic (UDP 137) from the same host:

![pfirewall.log with DROP entries from the Guest](screenshots/13-dc-firewall-log-drop-3389.png)

**Control test.** The DC itself can reach SMB on the IT client, which shows the port is open and the Guest's failure is caused by isolation, not by a closed port:

![DC to IT client on port 445 succeeds](screenshots/15-dc-to-it-445-control-test.png)

---

## Monitoring with Splunk

Splunk Enterprise runs on a Rocky Linux VM (not on the DC). A Universal Forwarder on the DC sends the Security, System and Application logs to port 9997 of the Splunk server, into an index named `wineventlog`.

**Data is arriving**, split by source:

```
index=wineventlog | stats count by host, source, sourcetype
```

![Events by host, source and sourcetype](screenshots/22-splunk-data-by-source.png)

```
index=wineventlog | timechart span=5m count by source
```

![Event volume over time](screenshots/23-splunk-event-volume.png)

**Guest logons to the domain.** Successful network logons (Logon Type 3) from the Guest machine, which ties the SIEM back to the isolation work:

```
index=wineventlog EventCode=4624 "10.10.50."
| table _time, host, Account_Name, Logon_Type, Source_Network_Address
```

![4624 logons from the Guest subnet](screenshots/24-splunk-guest-logons-4624.png)

**Failed logons (Event ID 4625)** grouped by source and account, the base query for a brute-force alert:

```
index=wineventlog EventCode=4625
| stats count by Source_Network_Address, Account_Name
| where count >= 5
```

![Failed logon search](screenshots/25-splunk-failed-logons-4625.png)

**Account and group changes:**

```
index=wineventlog EventCode IN (4720,4722,4724,4728,4732,4740)
| table _time, EventCode, host, Account_Name
```

![Account management events](screenshots/26-splunk-account-changes.png)

---

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| New NICs on the DC showed `AddressState: Invalid` | NIC was disconnected in VMware | Set to Connected, state became `Preferred` |
| IT scope created on `10.10.10.0` instead of `10.10.20.0` | Wrong start/end IP | Removed and recreated the scope |
| DNS option 006 pointed to `10.11.20.1` | Typo | Corrected to `10.10.20.1` |
| `nslookup` returned several IPs for the DC | Multi-homed DC registers every NIC in DNS | Disabled registration on the VLAN NICs and removed the `10.10.*` A records |
| The extra A records for `dc` came back later | The DC registered its NICs again | Re-checked `RegisterThisConnectionsAddress`, deleted the records again, then confirmed `dc` resolves only to `10.11.100.100` |
| `ipconfig /release` failed on the client | NIC had a static IP | Switched to DHCP |
| Client did not get an IP from its scope | NIC was on the wrong VMware network | Moved to the correct VMnet |
| Clock difference between client and DC | Different time zone (Kerberos rejects more than 5 minutes) | Fixed the time zone |
| Guest could ping the IT network | Forwarding was enabled on the Guest NIC and the IT client firewall was off | Disabled forwarding on VLAN50 |
| Guest could reach RDP on the DC after isolation | Windows Firewall was disabled on the DC | Enabled all profiles and applied the block rules |
| No `DROP` lines for some interfaces in the firewall log | All NICs were classified **Public** (likely because Network Location Awareness did not identify the domain at boot) and logging was on for the Domain profile only | Enabled `LogBlocked` on all three profiles. The block rules use `-Profile Any`, so isolation did not depend on the classification |
| Splunk search refused to run: minimum free disk space reached | Rocky VM disk was 80% full (3.5 GB free), below Splunk's 5000 MB default | Lowered `minFreeSpace` to 2000 in `server.conf` as a temporary fix, restarted Splunk. Permanent fix: expand the disk |
| Forwarder logged "output blocked" warnings | The indexer stopped indexing while the disk was below the limit | Fixed the disk limit, then confirmed data was arriving |
| `Restart-Service SplunkForwarder` failed | The service was already stopped | Started it with `splunk.exe restart` |

![Splunk refuses to search: minimum free disk space](screenshots/19-splunk-disk-space-error.png)

![minFreeSpace applied and Splunk health report](screenshots/20-splunk-minfreespace-fix.png)

![Forwarder warnings: TCP output blocked](screenshots/21-splunk-forwarder-blocked.png)

![All NICs classified as Public](screenshots/16-dc-network-category-public.png)

![DNS records for dc before cleanup](screenshots/17-dns-dc-records-duplicate.png)

![DNS records for dc after cleanup](screenshots/18-dns-dc-records-cleaned.png)

---

## Known limitations

- The DC doubles as router, DHCP and DNS server. In a real network these roles would be separate, with a dedicated firewall between segments.
- Windows Firewall filters traffic to the DC but not traffic routed between other segments, so isolation between HR, Sales, IT and Admins would need a real firewall (for example pfSense) between them.
- With forwarding disabled on the Guest NIC, the Guest has no Internet access.
- The Guest can still ping the DC (ICMP is not blocked).
- Any authenticated domain user can read AD and DNS data by default, so the Guest can still learn names and addresses even though it cannot connect to those hosts.
- The Splunk server's disk is small (17 GB), so the index size should be capped.

---

## Roadmap

- [x] Re-test Guest isolation after enabling the firewall and rules
- [x] Splunk: forwarder on the DC and Security log input
- [x] Search for failed logons (Event ID 4625)
- [ ] Save the 4625 search as an alert and trigger it with a brute-force test
- [ ] Add routes on ADC and CORE2 for `10.10.0.0/16` via `10.11.100.100`
- [ ] Test DHCP on VMnet10, VMnet12 and VMnet13
- [ ] Confirm the IT client is in the IT OU so its GPOs apply
- [ ] Remote Desktop GPO for IT machines
- [ ] Enable and tighten the firewall on the IT client
- [ ] Expand the Splunk server's disk and cap index sizes
- [ ] Replace the DC-as-router design with a firewall VM (pfSense) between segments
