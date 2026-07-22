# ESX-Kickstart-for-VCF-9.1-Lab
Kickstart ESX Installation for VSAN and Memory Tiering

# ESXi 9.x / VCF 9.1 Homelab Cluster Kickstart Blueprint

This repository contains a fully automated, sanitized, and modularized VMware ESXi 9.x / VMware Cloud Foundation (VCF) 9.1 Kickstart deployment template (`KS.cfg`). It is specifically optimized to provision a repeatable, high-performance homelab virtualization cluster on consumer-grade workstation platforms (such as the Minisforum MS-A2) utilizing AMD Ryzen processors.

## 🚀 Key Features & Automation Mechanics

- **Top-Level Variable Scoping:** All critical system properties (IPs, hostnames, drive paths, VLAN boundaries, MTUs) are centralized at the very top of the script inside a `# -------------------------` block.
- **Dynamic Pre-processing via %pre:** Utilizes a `%include` strategy to bypass native Kickstart variable limitations, dynamically compiling boot tracks onto the runtime file system before deployment.
- **Aggressive Drive Sanitization:** Automatically executes `partedUtil mklabel` and block-level `dd` operations to securely strip old vSAN storage signatures and metadata headers off secondary NVMe drives.
- **Rebootless NVMe Memory Tiering:** Implements the modern ESXi 9.x `esxcli memtier` API sequence inside host maintenance mode to successfully expand volatile memory pools.
- **Consumer Hardware Spoofing:** Employs precise `vmware/config` tweaks and CPUID brand string modifications to force compliance for community-supported nested vSAN ESA and NSX Edge infrastructure stacks.

---

## 🔧 Getting Started & Deployment Configuration

To spin up a new host, place the `KS.cfg` script on your network deployment share or boot media. Modify **only** the designated input block at the top of the file:

```text
# ---------------------------------------------------------
# [!] USER CONFIGURATION BLOCK - ONLY CHANGE THESE FIELDS [!]
# ---------------------------------------------------------
BOOT_DISK="t10.NVMe____GENERIC_BOOT_DISK_ID________________________0000000000000000"
VSAN_DISK="t10.NVMe____GENERIC_VSAN_DISK_ID________________________0000000000000000"
TIERING_DISK="t10.NVMe____GENERIC_TIERING_DISK_ID_____________________0000000000000000"

MANAGEMENT_IP="10.0.0.10"
NETMASK="255.255.255.0"
GATEWAY="10.0.0.1"
NAMESERVER="10.0.0.2"
HOSTNAME="esxi-host.local.lan"
DATASTORE_NAME="local-vmfs-datastore-01"
NTP_SERVER_IP="10.0.0.3"

V_MNIC="vmnic0"
MANAGEMENT_VLAN="10"
MANAGEMENT_VSWITCH_MTU="9000"
# ---------------------------------------------------------
```

---

## 🔍 Hardware Identification & Discovery Loop (Grabbing Device IDs)

When setting up a new host, you cannot run the full `KS.cfg` installer file until you know the exact serial string tags for your NVMe drives. To grab these cleanly without risking data loss or accidentally overwriting any drives, use a non-destructive **Discovery Loop**.

### 1. Create a `discovery.cfg` File
Save the following configuration block onto your bootable media or automated deployment network share as `discovery.cfg`. This script uses an emergency network fallback on native copper (`vmnic0`), runs entirely in volatile RAM, and deliberately pauses the installer so you can grab identifiers.

```text
vmaccepteula
network --bootproto=dhcp --device=vmnic0
rootpw VMware1!

%pre --interpreter=busybox
# Force start the micro-kernel internal runtime SSH service
/etc/init.d/SSH start

# Hold the installer environment completely open in an infinite sleep loop
while true; do
    sleep 60
done
```

### 2. Boot and Harvest the IDs
1. Temporarily connect an Ethernet cable to an onboard **RJ45 1G/2.5G copper port** (`vmnic0`) that has an active DHCP server on its segment. (Avoid high-speed 10G/100G SFP+ slots during discovery, as advanced drivers aren't fully live yet).
2. Boot your host using the `discovery.cfg` command flag at the boot loader (e.g., `ks=usb:/discovery.cfg`).
3. The ESXi installer console will halt permanently mid-boot at a prompt. This is expected behavior. Check your DHCP router lease tracking table to see what temporary IP address the host pulled.
4. Open a terminal on your workstation and SSH into the live BusyBox RAM-disk environment:
   ```bash
   ssh root@<THE_DHCP_IP_ADDRESS>
   ```
5. Authenticate using the password `VMware1!`. Because this is an active install memory layout rather than a fully compiled OS instance, standard `esxcli` commands will throw a `503 Service Unavailable` error. Instead, bypass the missing agent layer using either of these low-level approaches:

   * **Option A (Fastest Path):** Read the Linux sub-kernel device nodes directly from the disk path tracking tree to output all disk strings:
     ```bash
     ls -l /dev/disks/
     ```
   * **Option B (Engine Driver Bypass):** Force a direct programmatic query to the hypervisor driver system to print hardware models matched to serial tracking paths:
     ```bash
     localcli storage core device list | grep -E "Display Name:|Devfs Path:"
     ```

6. Copy the specific `t10.NVMe...` identifiers for your target **Boot Drive**, **vSAN Drive**, and **Memory Tiering Drive**.

Once you have recorded these IDs, you can close the terminal, plug your high-speed breakout DAC cables back in, populate the variable headers at the top of your master `KS.cfg` template, and fire up your final, fully automated cluster deployment!

---

## 🛠️ Post-Installation Verification Checklist

Once the automated deployment script triggers its final system reload sequence, verify cluster functionality:
- Log into your central vCenter environment and verify that total capacity reflects your expanded Pooled Memory Footprint (DRAM + NVMe Tier).
- Confirm that the targeted vSAN capacity drives have successfully formed pristine storage disk pools via the community-supported virtual environment drivers.

