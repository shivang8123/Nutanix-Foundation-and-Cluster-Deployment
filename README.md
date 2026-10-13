# Nutanix-Foundation-and-Cluster-Deployment

## 1. Purpose

It will cover Nutanix Foundation and cluster-deployment. Cover the deployment methods, virtualization concepts, network and IP prerequisites, Foundation workflow, post-deployment configuration, validation, testing, licensing, and handover.

> Always follow the Nutanix AOS/Foundation version-specific documentation and the approved customer design. Commands and UI labels can vary by release.

## 2. Ways to perform Foundation

Foundation can generally be performed using:

* **Foundation application** installed on a workstation or laptop.
* **Foundation VM**, deployed on a supported virtualization platform such as VirtualBox, VMware, or an existing Nutanix environment, depending on the Foundation version and deployment design.
* **Foundation Central or another centralized method**, when supported and selected for the customer environment.

The Foundation system must be able to reach both:

* The **IPMI/OOB network**: iDRAC, iLO, or IPMI.
* The **Nutanix management network**: AHV and CVM addresses.

If one network cannot route to the other, the Foundation VM may require two network interfaces. If one routed network can reach both, one interface may be sufficient.

The Foundation VM’s temporary IP address or addresses are not permanent cluster IPs. They are used only to perform imaging and deployment.

## 3. Virtualization and hypervisor concepts

### 3.1 Why use a hypervisor?

Without virtualization, one physical server normally runs one operating system directly. If separate Windows and Linux systems are required, additional physical servers may be needed. This can result in underutilized CPU, memory, and storage.

A hypervisor abstracts the physical hardware and allows multiple isolated virtual machines to run on the same server. Each VM can have its own operating system and allocated CPU, memory, storage, and network resources.

### 3.2 Hypervisor types

| Type | Description | Examples |
|---|---|---|
| Type 1 / bare-metal | Runs directly on the physical server hardware | Nutanix AHV, VMware ESXi, Microsoft Hyper-V |
| Type 2 / hosted | Runs as an application on an existing operating system | Oracle VirtualBox, VMware Workstation |


### 3.3 Nutanix AHV and CVM

In an AHV-based Nutanix cluster:

* AHV is the Type 1 hypervisor.
* VMs run on top of AHV.
* A **Controller VM (CVM)** runs on each Nutanix node.
* The CVM runs Nutanix AOS services and provides the software-defined storage functionality.
* The CVM manages the local disks and presents distributed storage services to the hypervisor and VMs.
* AHV uses a boot device for the hypervisor. The boot device is separate from the main storage devices used by the CVM and storage fabric.

A simplified I/O path is:

* CPU, memory, and some host operations are handled through the hypervisor and physical host.
* Storage I/O is serviced through the Nutanix CVM/AOS storage layer.

If a CVM becomes unavailable, storage services and VM behavior depend on the cluster state, resiliency, and workload. Do not assume that every VM will simply migrate automatically. Validate the behavior through the approved test plan.

## 4. IP address planning

### 4.1 Typical permanent IPs for a three-node cluster

For a three-node cluster, the normal permanent IP requirements are:

| Component | Quantity | Required? |
|---|---:|---|
| AHV host IPs | 3 | Yes |
| CVM IPs | 3 | Yes |
| IPMI/iDRAC/iLO IPs | 3 | Yes for out-of-band management |
| Cluster Virtual IP | 1 | Recommended/required for normal cluster access |
| Data Services IP | 1 | Optional, but commonly configured |
| Prism Central IP(s) | Separate | Required only when Prism Central is deployed |

Therefore, for a three-node cluster:

* **Management network:** normally 7 addresses if including 3 AHV IPs, 3 CVM IPs, and 1 Cluster VIP.
* **Add one Data Services IP** if the service is configured.
* **IPMI network:** 3 addresses, usually on a separate OOB network.
* **Prism Central:** plan separately according to the selected Prism Central deployment size and version.

**Correction:** Counting the Prism Central IP as one fixed cluster IP and stating a total of 12 or 13 is not universally correct. Prism Central is deployed separately and its IP requirements depend on the deployment model. The Cluster VIP and Data Services IP should also be distinguished clearly.

### 4.2 Network placement

Typically:

* AHV, CVM, Cluster VIP, and Data Services IP are placed on the Nutanix management network or on networks that satisfy the design requirements.
* IPMI/iDRAC/iLO can be on a separate OOB network and is preferably isolated from the management network.
* Prism Central must be reachable from the cluster and administrator/client networks according to the design.
* Gateway, subnet mask, DNS, and NTP details must be supplied by the customer’s network team.



### 4.3 Foundation VM addressing

Two common scenarios exist:

**Scenario A: Two Foundation VM interfaces**

* NIC 1 reaches the Nutanix management network.
* NIC 2 reaches the IPMI/OOB network.
* Each NIC receives an appropriate temporary IP address.

**Scenario B: One Foundation VM interface**

* One routed network can reach both the management and IPMI networks.
* One NIC and one IP address may be sufficient.

Before starting Foundation, test reachability from the Foundation system to all planned IPMI and management addresses.

## 5. Network terminology

### VLAN

A VLAN logically separates networks on the switching infrastructure. The VLAN ID must be confirmed by the customer’s network team.

* An **access port** carries traffic for one VLAN.
* A **trunk port** carries traffic for multiple VLANs.
* A **native VLAN** is carried untagged on a trunk, depending on the switch configuration.

If the switch is already handling the required untagged/native VLAN, the Nutanix-side VLAN value may be left as untagged or VLAN 0, depending on the product workflow. If the traffic must be tagged by Nutanix, enter the correct VLAN ID. Confirm the exact behavior with the switch design and the Foundation/AOS version.

### CIDR and subnet size

The `/24` notation is CIDR notation. It defines the network prefix length.

| Subnet | Total addresses | Normally usable host addresses |
|---|---:|---:|
| `/26` | 64 | 62 |
| `/25` | 128 | 126 |
| `/24` | 256 | 254 |
| `/23` | 512 | 510 |
| `/22` | 1,024 | 1,022 |


**Correction:** A `/24` has 256 total addresses and normally 254 usable host addresses—not 255 usable IPs.

## 6. Prerequisites

### 6.1 Customer prerequisites

Confirm that:

* Servers are delivered, racked, and powered on.
* Firmware and hardware compatibility meet the approved Nutanix requirements.
* IPMI/iDRAC/iLO access is available.
* Required server, storage, and network cables are connected.
* Switch ports are configured correctly as access or trunk ports.
* Required VLANs are allowed on the relevant switch ports.
* Management and IPMI networks are reachable according to the design.
* Required IP addresses are reserved and documented.
* Gateway, subnet mask, VLAN ID, DNS, and NTP details are available.
* IPMI usernames and passwords are available.
* The required AOS, AHV, Foundation, and firmware files are available.
* Required firewall rules are approved and implemented.

### 6.2 Foundation validation

From the Foundation application or VM:

* Ping or otherwise validate reachability to every IPMI/iDRAC/iLO address.
* Validate reachability to the planned AHV and CVM management network.
* Verify that no IP address conflicts exist.
* Log in to the IPMI interfaces and confirm that all required links and hardware are healthy.
* Confirm that all nodes are powered on and visible.

## 7. Foundation procedure

1. Prepare the Foundation application or deploy the Foundation VM.
2. Assign one or two temporary Foundation IP addresses, depending on routing.
3. Launch the Foundation UI.
4. Select the target nodes and provide the required IPMI credentials.
5. Upload or select the required AOS/AHV and Foundation images.
6. Enter the planned values:
   * AHV host IPs
   * CVM IPs
   * IPMI/iDRAC/iLO IPs
   * Subnet mask
   * Gateway
   * VLAN ID, if required
   * Cluster VIP
   * Data Services IP, if required
   * Cluster name, if supported during Foundation
7. Start the Foundation process and monitor each node.
8. Confirm that every node completes imaging successfully.
9. Confirm that AHV and CVM are reachable after imaging.

## 8. Post-Foundation procedure

### 8.1 Validate time on all AHV hosts and CVMs

Time consistency is important before cluster creation and during cluster operation.

On each AHV host and CVM, validate the system time with:

```bash
timedatectl
```

Check the system time, RTC/hardware time, timezone, and NTP synchronization status.

Use the approved NTP configuration wherever possible. If a manual correction is required during deployment, use the approved Linux procedure and then confirm the time again with `timedatectl`. The `hwclock` command can be used to inspect or synchronize the hardware clock when appropriate.


### 8.2 Create the cluster

Log in to one CVM with the approved Nutanix administrative account and run the version-appropriate cluster-creation command. A typical example is:

```bash
cluster -s <CVM_IP1>,<CVM_IP2>,<CVM_IP3> --redundancy_factor=2 create
```

Use the redundancy factor specified by the approved design. Do not select a value only because it was used in another environment.

### 8.3 Validate the cluster

Check the cluster status using the version-appropriate command, for example:

```bash
cluster status
```

Confirm that all CVMs and cluster services are healthy before continuing.

### 8.4 Configure AHV networking

After cluster creation, review the virtual switch and uplink configuration in Prism Element:

1. Open Prism Element.
2. Go to the network configuration or virtual switch section.
3. Confirm the expected uplinks on every host.
4. Confirm the bond mode required by the design, such as active-backup or a supported load-balancing mode.
5. Create or modify additional virtual switches only when required by the design.
6. Save the configuration and allow any rolling update/reboot to complete.
7. Confirm that all hosts remain healthy.

For CLI-level troubleshooting, commands such as the following may be available, depending on the AOS/AHV version:

```bash
manage_ovs show_interfaces
manage_ovs show_uplinks
```

Use the exact command syntax documented for the installed version.

### 8.5 Access Prism Element

Open:

```text
https://<CVM_IP>:9440
```

Use any healthy CVM IP. On first login:

* Change the default or initial password as prompted.
* Accept the End User License Agreement.
* Configure Nutanix Pulse according to the customer’s policy.
* Set the cluster name.
* Configure or confirm the Cluster VIP and Data Services IP.
* Configure the required storage container(s).
* Remove or retain default containers according to the approved design.
* Configure DNS and NTP.
* Configure HA/host-reservation and capacity-reservation settings according to the design.
* Review alerts and cluster health.

### 8.6 Create networks and deploy Prism Central

1. Create or update the required virtual switches.
2. Create the required AHV subnets and VLAN-backed networks.
3. Upload the required Prism Central image, if applicable.
4. Deploy Prism Central using the approved sizing and IP plan.
5. Register or attach the Prism Element cluster to Prism Central.
6. If the customer already has an approved Prism Central, use that environment instead of deploying another one.

## 9. Day-2 configuration

Perform only the tasks included in the customer statement of work and design. Common activities include:

* Active Directory or other directory-service integration.
* Syslog integration.
* SMTP configuration.
* Alert and notification configuration.
* Image upload, such as an ISO or QCOW2 image.
* Test VM creation.
* Required subnet and security configuration.
* Licensing.

## 10. Test plan and handover

Before handover, execute the approved test plan. Typical checks include:

* All expected nodes and CVMs are healthy.
* Cluster services are running.
* Storage containers and subnets are available.
* A test VM can be created, powered on, and accessed.
* VM network connectivity works as expected.
* Host and CVM failure scenarios behave according to the design.
* Maintenance-mode behavior is validated.
* Alerts and monitoring integrations work.
* DNS, NTP, SMTP, directory, and syslog integrations are validated where applicable.
* The as-built document is completed and reviewed.
* Cluster licensing is applied.

The handover package should normally include:

* Cluster and node details.
* AHV, CVM, IPMI, Cluster VIP, and Prism Central information.
* Network and VLAN details.
* Storage container and subnet details.
* Test-plan results.
* As-built documentation and diagrams.
* Operational credentials or secure credential-transfer instructions, according to policy.
* Knowledge-transfer material for the customer’s administrators.

