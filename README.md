# ESX-Kickstart-for-VCF-9.1-Lab
Kickstart ESX Installation for VSAN and Memory Tiering

# ESXi 9.x / VCF 9.1 Homelab Cluster Kickstart Blueprint

This repository provides an automated, variablized, and repeatable bare-metal provisioning blueprint for VMware ESXi 9.x Hosts designed for VMware Cloud Foundation (VCF) 9.1. It is tailored specifically for deploying reliable, fast, virtualization clusters on consumer workstation architectures, including small-form-factor AMD Ryzen nodes.

## 🤝 Project Credits & Acknowledgement

This deployment template is designed to natively complement the excellent [VMware Cloud Foundation (VCF) 9.1 in a Box](https://github.com) project developed by **William Lam**. 

While William's automation excels at orchestrating the virtualized management environment components, this repository streamlines physical node readiness. It delivers a highly modular, single-pane user configuration interface to easily "repave" hardware platforms, forcing absolute compliance even if drives contain active local partitions or stubborn cluster flags.

### Included Enhancements from William Lam's Guides
This solution incorporates the consumer hardware optimizations and cluster performance properties from William's sample KS-ESXxx.CFG Files directly into a touchless Kickstart routine, including:
- **AMD Ryzen System Stability Tuning:** Implements vital hypervisor core configurations (`disable_apichv = "TRUE"`) to avoid hardware-level scheduling deadlocks.
- **vSAN ESA Mock Hardware Integration:** Automatically pulls, registers, and provisions William Lam's custom `nested-vsan-esa-mock-hw.vib` repository packages to bypass consumer NVMe storage abstraction boundaries.
- **Low-Level Platform Overrides:** Injects automated CPUID brand masking strings (`cpuid.brandstring = "AMD EPYC Ryzen 9 9955HX"`) to ensure nested infrastructure layers like NSX Edges initialize flawlessly on non-EPYC architectures.
- **Advanced Resource Management Optimizations:** Sets enterprise memory sharing (`ShareForceSalting`) and vSAN I/O traffic throttles (`DOMNetworkSchedulerThrottleComponent`) for small labs.

---

## 📁 Repository Structure

- [`KS.cfg`](./KS.cfg) — The production installation script featuring a centralized variable block.
- [`harvest.cfg`](./harvest.cfg) — Non-destructive deployment discovery engine to gather raw NVMe device tracking variables.

---

## 🔍 Step 1: Device Harvesting (Before Installation)

When configuring a host, the main installation script requires the exact hardware serial tracking strings of your NVMe drives. To gather these cleanly without risking data loss, use the non-destructive discovery loop.

1. Download [`harvest.cfg`](./harvest.cfg) and place it on your bootable deployment media or network share.
2. Connect an Ethernet cable from a temporary DHCP network switch directly into one of the native onboard **RJ45 1G/2.5G copper ports** (`vmnic0`) on the host. 
3. Boot the machine using the discovery configuration (e.g., passing `ks=usb:/harvest.cfg` at the boot loader).
4. The system will load the installer into RAM and halt permanently at a blank screen. Check your network DHCP lease table to find the temporary IP address the host pulled.
5. Open a terminal on your workstation and SSH into the live installer environment:
   ```bash
   ssh root@<THE_DHCP_IP_ADDRESS>
   ```
6. Authenticate using the temporary password: `VMware1!`. Run either of these commands to dump your raw storage identifiers directly out of the micro-kernel abstraction layer:
   ```bash
   # Approach A: Target device nodes directly
   ls -l /dev/disks/

   # Approach B: Engine driver bypass query
   localcli storage core device list | grep -E "Display Name:|Devfs Path:"
   ```
7. Copy and record the unique strings starting with `t10.NVMe...` for your **Boot Drive**, **vSAN Drive**, and **Memory Tiering Drive**. You can now power down the host.

---

## 🔧 Step 2: Production Deployment Configuration

Once you have recorded your device IDs, you can execute your production deployment.

1. Download [`KS.cfg`](./KS.cfg).
2. Open the file in a text editor and fill out the centralized **User Configuration Block** located right at the top of the script (Lines 10–23):
   ```text
   BOOT_DISK="t10.NVMe____YOUR_BOOT_DISK_ID..."
   VSAN_DISK="t10.NVMe____YOUR_VSAN_DISK_ID..."
   TIERING_DISK="t10.NVMe____YOUR_TIERING_DISK_ID..."
   
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
   ```
3. Re-connect your high-speed SFP+ or breakout network infrastructure cabling to the host.
4. Boot the machine from your modified production script (e.g., `ks=usb:/KS.cfg`). The script will automatically clean stale partitions, deploy the OS, configure static networking, clear sticky metadata flags, and bind your memory tier natively via the correct ESXi 9.x `esxcli memtier` API.

## 🛠️ Post-Installation Verification Checklist

Once the automated deployment script triggers its final system reload sequence, verify cluster functionality:
- Log into your central vCenter environment and verify that total capacity reflects your expanded Pooled Memory Footprint (DRAM + NVMe Tier).
- Confirm that the targeted vSAN capacity drives have successfully formed pristine storage disk pools via the community-supported virtual environment drivers.
