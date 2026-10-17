---
title: "Installing a Windows Server VM on Proxmox VE"
description: "Step by step: setting up a Windows Server VM on Proxmox VE, including VirtIO drivers and the QEMU guest agent."
date: 2026-10-17T09:27:31+02:00
lastmod: 2026-10-17T09:27:31+02:00
draft: false
categories:
  - tutorial
tags:
  - proxmox
  - windows-server
  - virtio
  - virtualization
series: []
showDate: true
showDateUpdated: true
showReadingTime: true
showTableOfContents: true
showWordCount: true
showAuthor: true
showTaxonomies: true
showPagination: true
---

The following describes the standard way to install a Windows Server VM on Proxmox (Virtual Environment) - which we will call PVE from here on. The screenshots are from an installation of Windows Server 2022 on PVE 8.4. With Windows Server 2025 and PVE 9 the process is the same; where something differs, it is mentioned.

## Overview

1. Downloading the Windows Server and VirtIO driver ISOs
2. VM configuration
3. OS installation and drivers after the installation

## References

- [Windows Server 2025 download](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025)
- [Windows Server 2022 download](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- [Windows VirtIO Drivers](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)
- [Windows 2022 guest best practices](https://pve.proxmox.com/wiki/Windows_2022_guest_best_practices)

## 1. Downloading the Windows Server and VirtIO Driver ISOs

This guide uses Windows Server 2022 for the installation. The current version 2025 can be downloaded from this [link](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025); 2022 is in the references.
In addition you need an ISO with the VirtIO drivers for the Windows installation, so that we can install the storage and network drivers, the guest agent, etc. on the Windows Server. Information about the ISO can be found in Proxmox' official [documentation](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers). Take the current "stable" ISO from there.

Once the download is finished, the ISO files have to be stored on the PVE host, see the next image.
Click on the storage on the left that holds your ISO files. Go to 'ISO Images' and upload the ISOs you just downloaded to PVE:

![ISO upload in PVE](pve_iso_upload.png)

## 2. VM Configuration

In the next step we set up the VM in PVE on which we will install the Windows Server.

1. Click on "Create VM" in the top right corner of the web UI:

![Create VM](pve_vm_part1.png)

2. Give the VM a 'VM ID' - it has to be unique - and a name:

![General](pve_vm_part2.png)

3. In the 'OS' tab select your downloaded Windows Server ISO as the ISO image, see the next image. Select "Microsoft Windows" as the guest OS type and the matching version, here '11/2022/2025'. On the right you can also tick 'Add additional drive for VirtIO drivers' to attach the VirtIO drivers directly; select the VirtIO ISO you just downloaded, see image:

![OS](pve_vm_part3.png)

4. Set the configuration as shown in the next image: machine 'q35', BIOS 'OVMF (UEFI)' with an EFI disk, TPM v2.0 and 'Qemu Agent' ticked:

![System](pve_vm_part4.png)

5. Disks: this is where the VM's disk is created. I used 'VirtIO Block' as the bus, with 'Discard' and 'IO thread' ticked. For Windows, Proxmox nowadays recommends 'SCSI' as the bus with the 'VirtIO SCSI single' controller. Both work; the only thing that matters is that you load the matching driver during the installation later (see section 3).

![Disks](pve_vm_part5.png)

6. CPU: one socket with 2 cores, type 'host'. If the VM is going to be migrated between hosts with different CPUs later, use a generic type like 'x86-64-v2-AES' instead of 'host'.

![CPU](pve_vm_part6.png)

7. Memory: 4096 MiB.

![Memory](pve_vm_part7.png)

8. Network: bridge 'vmbr0', model 'VirtIO (paravirtualized)'.

![Network](pve_vm_part8.png)

9. Confirm: check the summary and click 'Finish'.

![Confirm](pve_vm_part9.png)

## 3. OS Installation

1. In the next step we start the VM and open the console. On the first boot the Windows ISO briefly shows "Press any key to boot from CD or DVD"; you have to press a key quickly. If you miss it, you end up in the UEFI boot manager or the UEFI shell. In that case restart the machine via 'Reset' and press a key in time. After that you see the familiar Windows setup. Click 'Next' and then 'Install now':

![Windows setup](pve_os_part1.png)

2. Select the Windows Server operating system you want. We choose: 'Windows Server 2022 Standard Evaluation (Desktop Experience)'. Without '(Desktop Experience)' only Server Core without a graphical interface is installed. Accept the license terms in the next step.

![Select the operating system](pve_os_part2.png)

3. In the next step we do the partitioning; here we first have to load the VirtIO storage driver, because otherwise Windows does not see the VirtIO disk. Select the 'Custom' method:

![Custom installation](pve_os_part3a.png)

In the next step click on 'Load driver':

![Load driver](pve_os_part3b.png)

Next, select the driver that matches your OS. In our case it is the one for version 2022 ('2k22'). For Windows Server 2025 use '2k25' accordingly. The list shows 'viostor' here because the disk was created as 'VirtIO Block'; for a 'SCSI' disk the driver is called 'vioscsi':

![Select the driver](pve_os_part3c.png)

Afterwards the disk should be shown:

![Disk](pve_os_part3d.png)

Select the disk and start the installation.

4. When the installation is done, the server should restart and boot into your new Windows Server. You are asked to set a new password for the local administrator.

![Administrator password](pve_os_part4.png)

5. Now you can log in. You send Ctrl+Alt+Delete via the button on the left of the noVNC console:

![Login](pve_os_part5.png)

6. Driver installation. Log in to your new server, open the File Explorer and open the drive with the still attached VirtIO driver ISO:

![VirtIO ISO in the Explorer](pve_os_part6.png)

Install the 'virtio-win-gt-x64' drivers with a double click.
After that the server should have a network connection. Restart the server afterwards.

7. We also install the QEMU guest agent: in the Explorer, open the 'guest-agent' folder on the VirtIO ISO and install 'qemu-ga-x86_64.msi':

![Guest agent](pve_os_part7.png)

Alternatively, 'virtio-win-guest-tools.exe' in the root of the ISO installs the drivers and the guest agent in one step.

Afterwards the server's IP addresses should be shown in the PVE web UI:

![IP addresses in PVE](pve_os_part8.png)

Finally, you can remove both ISOs from the VM under 'Hardware'.

## Differences with Windows Server 2025

- The Windows Server 2025 setup looks different from the screenshots above. The steps are the same: choose the edition, 'Load driver', load the driver from '2k25', select the disk, install.
- In PVE 9 the process of creating the VM is the same; as the OS version you still select '11/2022/2025'.
- The evaluation version runs for 180 days. For production use you need a regular license.
