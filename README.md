# Enterprise Active Directory Infrastructure & Endpoint Governance Lab

---

## Executive Summary

This enterprise simulation project designs, implements, and secures an isolated on-premises enterprise identity infrastructure using **Windows Server 2022** and **Windows 10 Pro**. The primary objective was establishing a resilient directory baseline adhering to the **Principle of Least Privilege (PoLP)**, configuring hierarchical Organizational Units (OUs), enforcing centralized endpoint compliance via **Group Policy Objects (GPO)**, and resolving real-world hypervisor and operating system integration challenges.

---

## Infrastructure Architecture & Network Topology

The environment is deployed in an air-gapped VirtualBox **Internal Network** (`AD-Lab`) to emulate a private enterprise subnet without reliance on external DHCP or public DNS resolution.

| Parameter | Domain Controller (`DC-01`) | Client Workstation (`CL-WIN10`) |
|---|---|---|
| **Operating System** | Windows Server 2022 Datacenter | Windows 10 Pro (x64) |
| **Assigned Role** | Root Forest DC, DNS Server, Kerberos KDC | Domain Member Endpoint |
| **FQDN** | `DC-01.corp.local` | `CL-WIN10.corp.local` |
| **IP Address** | `192.168.10.10` | `192.168.10.20` |
| **Subnet Mask** | `255.255.255.0` (`/24`) | `255.255.255.0` (`/24`) |
| **Default Gateway** | None (Isolated Subnet) | None (Isolated Subnet) |
| **Primary DNS** | `127.0.0.1` (Loopback) | `192.168.10.10` (Points to `DC-01`) |
| **Virtual Hardware** | 2 vCPUs, 4096 MB RAM, 50 GB VDI | 2 vCPUs, 4096 MB RAM, 50 GB VDI |

---

## Implementation Lifecycle

## Phase 1: Domain Controller Baseline & Forest Deployment

### System Initialization

Configured static network binding on interface `Ethernet0` and updated server NetBIOS hostname to `DC-01`.

### Role Deployment

Installed **Active Directory Domain Services (AD DS)** and integrated **DNS Server** roles via Server Manager.

### Forest Promotion

Promoted `DC-01` to the primary domain controller for the new forest `corp.local` with Windows Server 2016 Forest/Domain Functional Levels (FDFL/DDFL), configuring the Directory Services Restore Mode (DSRM) credential database.

---

## Phase 2: Directory Hierarchy & Role-Based Access Control (RBAC)

### Root OU Architecture

Designed a production-standard OU tree under `_CORP_HQ` to segment user identities, system tiers, and administrative scopes:

```text
_CORP_HQ
├── Administration
├── HR_Department
├── IT_Department
├── Finance
├── Workstations
└── Groups
```

### OU Purpose Breakdown

| OU | Purpose |
|---|---|
| `_CORP_HQ\Administration` | Dedicated admin accounts and service principals |
| `_CORP_HQ\HR_Department` | Human Resources personnel identities |
| `_CORP_HQ\IT_Department` | System administration and helpdesk personnel |
| `_CORP_HQ\Finance` | Accounting and financial records staff |
| `_CORP_HQ\Workstations` | Corporate computer objects |
| `_CORP_HQ\Groups` | Global and Domain Local Security Groups |

### Identity & Group Provisioning

* Created user identity `Jane.Smith` in `HR_Department`.
* Assigned UPN:

```text
Jane.Smith@corp.local
```

* Configured Global Security Group:

```text
SG_HR_Users
```

* Assigned `Jane.Smith` as an active member to establish role-based delegation.

---

## Phase 3: Workstation Integration & Domain Join

### Static DNS Binding

Aligned client network adapter `Ethernet` to DNS resolver:

```text
192.168.10.10
```

This enabled SRV record resolution for:

```text
_ldap._tcp.dc._msdcs.corp.local
```

### Domain Trust Establishment

Executed an authenticated domain join to:

```text
corp.local
```

The workstation joined using Domain Administrator credentials, creating a secure machine trust account in Active Directory.

### Object Management

Relocated the computer object `CL-WIN10` from the default `CN=Computers` container into:

```text
OU=Workstations,OU=_CORP_HQ,DC=corp,DC=local
```

This enabled granular GPO inheritance.

---

## Phase 4: Group Policy Engineering & Endpoint Enforcement

### Policy Architecture

Created and linked the following GPO directly to the HR Department OU:

```text
GPO_HR_Security_Baseline
```

Linked location:

```text
OU=HR_Department,OU=_CORP_HQ,DC=corp,DC=local
```

### Administrative Lockdown Configuration

Configured the following policy path:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
            └── Prohibit access to Control Panel and PC settings
```

Policy setting:

```text
Enabled
```

### Client Convergence

Logged into `CL-WIN10` as `Jane.Smith`, refreshed policy cache, and confirmed policy processing.

Commands used:

```powershell
gpupdate /force
gpresult /r
```


---

## Technical Challenges Faced & Root-Cause Engineering Solutions

## 1. VirtualBox EFI/TPM Emulation Boot Hang

### Symptom

Client VM remained suspended on a blank black display upon power-on when provisioning Windows 11 under UEFI mode.

### Root Cause Analysis

VirtualBox 7.x UEFI firmware initializes an aggressive 1.5-second boot timeout window for non-volatile optical media. If the keypress interrupt fails to register or if hypervisor video acceleration handshake lags, execution halts at a black screen without falling back to BIOS text prompts.

### Engineering Resolution

Replaced the UEFI Windows 11 configuration with a standardized Windows 10 enterprise deployment utilizing legacy BIOS execution mode:

```text
PIIX3 chipset
No EFI
```

This eliminated hypervisor abstraction latency, enabling deterministic boot sequences.

---

## 2. VirtualBox Unattended Installation ISO Lock

### Symptom

The VM continually booted into an empty VirtualBox unattended installation wrapper image rather than the target Windows installer.

### Root Cause Analysis

The hypervisor automounted an internal `Unattended.vdi` auxiliary image on SATA Port 1, preempting the main OS installation ISO.

### Engineering Resolution

Dismounted the unattended image via VirtualBox Storage Manager, remounted the verified ISO image on SATA Port 0, and checked the **Skip Unattended Installation** parameter during VM initialization.

---

## 3. Windows 10 Home Edition Domain Join Restriction

### Symptom

The **Domain** radio button in `sysdm.cpl` was entirely disabled/greyed out with the warning:

```text
You cannot join a computer running this edition of Windows 10 to a domain.
```

### Root Cause Analysis

The automated installer selected the Windows 10 Home product image by default, which lacks the underlying Active Directory client binaries, Netlogon services, and Kerberos domain integration capabilities.

### Engineering Resolution

Performed an instant in-place operating system upgrade without wiping or reinstalling.

Navigation path:

```text
Settings
└── Update & Security
    └── Activation
        └── Change product key
```

Injected Microsoft’s generic Pro KMS channel key:

```text
VK7JG-NPHTM-C97JM-9MPGT-3V66T
```

Rebooted the system. This converted the host to **Windows 10 Pro**, unlocking the domain-join subsystem.

---

## 4. First-Time Domain User Logon Initialization Latency

### Symptom

Initial interactive logon for `Jane.Smith` displayed a persistent **Welcome** loading spinner for several minutes.

### Root Cause Analysis

Because the virtual machines were placed on an isolated internal subnet with no internet access or external gateway, the Windows profile subsystem executed background timeout routines trying to poll Microsoft cloud telemetries and Windows Update endpoints during initial local profile creation:

```text
C:\Users\Jane.Smith
```

### Engineering Resolution

Allowed the system network stack to complete its default timeout sequence, successfully falling back to the local Kerberos Key Distribution Center (KDC) ticket on `DC-01` and loading the domain user desktop profile cleanly.

---

## Security & Governance Highlights

### Principle of Least Privilege

End-users are strictly assigned Standard Domain User privileges. Administrative tasks are restricted to delegated credentials within the `Administration` OU.

### Centralized Configuration Management

Critical operating system binaries and administrative entry points such as Control Panel and Settings applets are locked down across non-administrative OUs via GPO, mitigating lateral movement and tampering risks.

### Kerberos v5 Authentication

All client-to-server traffic uses mutual authentication handled natively by the AD DS Key Distribution Center (KDC) over TCP/UDP Port 88.

---

## Verification Screenshots & Technical Artifacts

Below is the verified photographic evidence demonstrating each completed milestone of the Active Directory and Group Policy implementation lifecycle.

| Artifact | File Reference | Technical Milestone Verified |
|---|---|---|
| **Milestone 01** | `01-dc-ip-config.png` | Static IP (`192.168.10.10`), loopback DNS, and NetBIOS name (`DC-01`) baseline |
| **Milestone 02** | `02-ad-promotion.png` | Successful AD DS role installation and forest creation for `corp.local` |
| **Milestone 03** | `03-ou-hierarchy.png` | Scalable OU tree under `_CORP_HQ`, user object `Jane.Smith`, and `SG_HR_Users` group |
| **Milestone 04** | `04-domain-join-success.png` | Workstation FQDN confirmation (`CL-WIN10.corp.local`) and Active Directory trust handshake |
| **Milestone 05** | `05-gpo-enforcement-proof.png` | Group Policy application (`GPO_HR_Security_Baseline`) displayed in `gpresult /r` alongside real-time OS access denial dialog |

---

### Milestone 1: Domain Controller Network & Hostname Configuration

**Artifact Description:** Static IPv4 address binding (`192.168.10.10/24`), loopback DNS configuration (`127.0.0.1`), and persistent NetBIOS system hostname (`DC-01`).

**Technical Validation:** Demonstrates establishing a solid networking baseline required prior to AD DS promotion to prevent dynamic IP lease dropouts and DNS replication failures.

<img width="1001" height="723" alt="01-dc-ip-config" src="https://github.com/user-attachments/assets/29d9c312-c761-4bff-90da-403aab1aea7a" />


---

### Milestone 2: AD DS Role Installation & Forest Root Promotion

**Artifact Description:** Deployment of the Active Directory Domain Services role and successful forest root creation for the namespace `corp.local`.

**Technical Validation:** Verifies the deployment of the centralized Kerberos KDC, Schema partition, Global Catalog, and integrated DNS forward lookup zones for enterprise directory queries.

<img width="745" height="564" alt="02-ad-promotion" src="https://github.com/user-attachments/assets/e01b64a5-6274-407a-b737-a0f27f117b13" />


---

### Milestone 3: Organizational Unit Hierarchy & RBAC Architecture

**Artifact Description:** Structured tree under root OU `_CORP_HQ`, displaying segmented sub-OUs (`Administration`, `HR_Department`, `IT_Department`, `Finance`, `Workstations`, `Groups`), alongside the provisioned user identity `Jane.Smith` and global security group `SG_HR_Users`.

**Technical Validation:** Proves implementation of Role-Based Access Control (RBAC) and clean object compartmentalization adhering to the Principle of Least Privilege (PoLP) for scalable Group Policy inheritance.

<img width="753" height="528" alt="03-ou-hierarchy" src="https://github.com/user-attachments/assets/1a81c6c5-5cfd-45d8-b14a-c9e513559953" />


---

### Milestone 4: Workstation Domain Join & Machine Trust Handshake

**Artifact Description:** System Properties verification window on `CL-WIN10` showing `Full computer name: CL-WIN10.corp.local` and active membership in domain `corp.local`.

**Technical Validation:** Confirms end-to-end DNS SRV record resolution across the isolated virtual network, authenticated Kerberos exchange with `DC-01`, and the generation of a secure computer account object within Active Directory.

<img width="323" height="390" alt="04-domain-join-success" src="https://github.com/user-attachments/assets/7807a551-b51d-4d87-af3f-487216c13d66" />


---

### Milestone 5: Centralized Group Policy (GPO) Enforcement Proof

**Artifact Description:** Split-view showing the output of `gpresult /r` in Command Prompt confirming `GPO_HR_Security_Baseline` applied to user `Jane.Smith`, alongside the native Windows restriction dialog:

```text
This operation has been cancelled due to restrictions in effect on this computer.
```

**Technical Validation:** Proves end-to-end policy authoring, OU linking, client-side policy retrieval, and effective endpoint hardening against unauthorized administrative system access.

<img width="1011" height="783" alt="05-gpo-enforcement-proof" src="https://github.com/user-attachments/assets/977213d4-bbd3-40e8-81f1-d8b53102f954" />


---

## Technical Skills & Tools Demonstrated

### Directory Services

* Active Directory Domain Services (AD DS)
* Group Policy Management (GPMC)
* Active Directory Users and Computers (ADUC)
* Directory Services Restore Mode (DSRM)
* Organizational Unit Architecture
* Kerberos Authentication

### Systems Administration

* Windows Server 2022
* Windows 10 Pro
* Sysprep
* Generic Volume License Key (GVLK) Elevation
* In-place OS Upgrades

### Networking & Protocols

* DNS Forward Lookup Zones
* SRV Records
* IPv4 Static Addressing
* Subnetting
* TCP/IP Stack Troubleshooting

### Diagnostics & Utilities

```powershell
gpupdate /force
gpresult /r
ping
ipconfig /all
sysdm.cpl
ncpa.cpl
```

### Hypervisor & Virtualization

* Oracle VM VirtualBox
* Internal Network Sandboxing
* Virtual Hardware Allocation
* UEFI vs. Legacy BIOS Firmware Troubleshooting


---

## Future Improvements & Scalability Roadmap

### Automated Identity Lifecycle Management

Develop an end-to-end PowerShell provisioning script utilizing `New-ADUser` to parse employee `.csv` directories, automatically generate secure randomized passwords, and populate department-specific OUs.

### Dynamic IP Allocation & DNS Scopes

Deploy and authorize the **DHCP Server** role on `DC-01`, configuring the following scope options for zero-touch client onboarding:

```text
003 Router/Gateway
006 DNS Servers
015 Domain Name
```

### Tiered Tier-0/Tier-1 Administration Model

Implement Active Directory administrative tiering such as ESA/Red Forest concepts to prevent domain-level credentials from touching member workstations.

### Departmental Network Storage

Deploy file services with **Access-Based Enumeration (ABE)** and granular NTFS permissions mapped via GPO Drive Maps.



