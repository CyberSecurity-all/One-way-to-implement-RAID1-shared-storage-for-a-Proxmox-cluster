# One-way-to-implement-RAID1-shared-storage-for-a-Proxmox-cluster
The following solution, in my opinion, is simple and cheap to implement.

<div align="center">
<img src="shared storage for a Proxmox cluster.png" style="width: 500px;height:500px" alt="One-way-to-implement-RAID1-shared-storage-for-a-Proxmox-cluster">
</div>
<br>
<br>

**It consists of the following:**  

1. There are two nodes with nvme disks (not necessary). On each installed package: "nvme-cli" ("nvme-tcp", we broadcast disks to the network from each node, you can also use the icsi package "tgt") and package: "open-iscsi", package: "multipath-tools" (as one of the methods, can be used to improve performance, you can bond, when it is possible to connect nodes with several network connections).  

2. A network is established between these nodes.  

3. On one of the nodes we create a virtual machine “VSAN105” based on Linux with approximately the following characteristics: RAM 512Mb, HDD 3Gb and install the packages: “mdadm” (we will build a RAID1 array from the corresponding sections of the nodes’ nvme disks: nvme discovery…, connect…, and then support it), “tgt” (for broadcasting the md0 iscsi target to two nodes), “lvm2” (from the resulting md0 array we do pvcreate, vgcreate).  

4. Configure "open-iscsi" on the nodes.  

5. Launch VSAN. It broadcasts iscsi md0 with the corresponding VG.  

6. In the Proxmox data center we create a shared LVM storage.  

7. Make this virtual machine work in RAM without using disk. Now it can work in live migration mode and support shared storage.  

8. We transfer the VSAN virtual machine to HA.  

9. Notification, diagnostics, and solution of RAID problems are the responsibility of the utilities of the “mdadm” package.
    
## 1. Preparation of nodes.
### 1.1. There are two nodes with nvme disks. On each, the package: "nvme-cli" is installed, and the package: "open-iscsi", the package: "multipath-tools":
```
apt update 
apt install nvme-cli open-iscsi multipath-tools  
```
### 1.2. Broadcast disks to the network from each node:
```
On node pve1 10.10.1.1  
```
**Let's make an executable file:**  
```
nano  /root/nvme_start.sh
#!/bin/bash
modprobe nvmet
modprobe nvmet-tcp
# Port settings
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1
echo 10.10.1.1 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.1.1 tcp ipv4"
mkdir /sys/kernel/config/nvmet/ports/2
cd /sys/kernel/config/nvmet/ports/2
echo 10.10.2.1 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.2.1 tcp ipv4"
name_nvme_disk=$(udevadm info --query=name /dev/disk/by-path/pci-0000:02:00.0-nvme-1)
echo "$name_nvme_disk"
        cd /sys/kernel/config/nvmet/subsystems
        mkdir ora10
        cd ora10
        echo "oral0"
        echo -n 1 > attr_allow_any_host
        nvme list | grep -w $name_nvme_disk | awk '{print "pve1-"$3}' > attr_serial
        nvme list | grep -w $name_nvme_disk | awk '{print "pve1-"$4}' > attr_model
        cd namespaces/
        mkdir "10"; cd "10"
        echo -n "/dev/${name_nvme_disk}" > device_path; echo -n 1 > enable;
        # Кожну підсистему, опісля конфигурації, приєднуємо до порту
        sleep 5
        ln -s /sys/kernel/config/nvmet/subsystems/ora10 /sys/kernel/config/nvmet/ports/1/subsystems/ora10
        ln -s /sys/kernel/config/nvmet/subsystems/ora10 /sys/kernel/config/nvmet/ports/2/subsystems/ora10
exit 0  
```
```
root@pve1:~# ./root/nvme_start.sh
root@pve1:~# apt install tree

root@pve1:~#  tree /sys/kernel/config/nvmet/ 
/sys/kernel/config/nvmet/
├── hosts
├── ports
│   ├── 1
│   │   ├── addr_adrfam
│   │   ├── addr_traddr
│   │   ├── addr_treq
│   │   ├── addr_trsvcid
│   │   ├── addr_trtype
│   │   ├── addr_tsas
│   │   ├── ana_groups
│   │   │   └── 1
│   │   │       └── ana_state
│   │   ├── param_inline_data_size
│   │   ├── param_pi_enable
│   │   ├── referrals
│   │   └── subsystems
│   │       └── ora10 -> ../../../../nvmet/subsystems/ora10
│   └── 2
│       ├── addr_adrfam
│       ├── addr_traddr
│       ├── addr_treq
│       ├── addr_trsvcid
│       ├── addr_trtype
│       ├── addr_tsas
│       ├── ana_groups
│       │   └── 1
│       │       └── ana_state
│       ├── param_inline_data_size
│       ├── param_pi_enable
│       ├── referrals
│       └── subsystems
│           └── ora10 -> ../../../../nvmet/subsystems/ora10
└── subsystems
    └── ora10
        ├── allowed_hosts
        ├── attr_allow_any_host
        ├── attr_cntlid_max
        ├── attr_cntlid_min
        ├── attr_firmware
        ├── attr_ieee_oui
        ├── attr_model
        ├── attr_pi_enable
        ├── attr_qid_max
        ├── attr_serial
        ├── attr_version
        ├── namespaces
        │   └── 10
        │       ├── ana_grpid
        │       ├── buffered_io
        │       ├── device_nguid
        │       ├── device_path
        │       ├── device_uuid
        │       ├── enable
        │       ├── p2pmem
        │       └── revalidate_size
        └── passthru
            ├── admin_timeout
            ├── clear_ids
            ├── device_path
            ├── enable
            └── io_timeout

21 directories, 41 files
root@pve1:~#   
```
**On node pve99 10.10.1.2**  

**Let's make a similar executable file:**  
```
nano  /root/nvme_start.sh 
#!/bin/bash
modprobe nvmet
modprobe nvmet-tcp
# # Налаштування порта
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1
echo 10.10.1.2 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.1.2 tcp ipv4"
mkdir /sys/kernel/config/nvmet/ports/2
cd /sys/kernel/config/nvmet/ports/2
echo 10.10.2.2 | tee -a addr_traddr > /dev/null
echo tcp | tee -a addr_trtype > /dev/null
echo 4420 | tee -a addr_trsvcid > /dev/null
echo ipv4 | tee -a addr_adrfam > /dev/null
echo "port 4420 IP 10.10.2.2 tcp ipv4"
name_nvme_disk=$(udevadm info --query=name /dev/disk/by-path/pci-0000:05:00.0-nvme-1)
echo "$name_nvme_disk"
        cd /sys/kernel/config/nvmet/subsystems
        mkdir ora20
        cd ora20
        echo "ora20"
        echo -n 1 > attr_allow_any_host
        nvme list | grep -w $name_nvme_disk | awk '{print "pve99-"$3}' > attr_serial
        nvme list | grep -w $name_nvme_disk | awk '{print "pve99-"$4}' > attr_model
        cd namespaces/
        mkdir "20"; cd "20"
        echo -n "/dev/${name_nvme_disk}" > device_path; echo -n 1 > enable;
        # Кожну підсистему, опісля конфигурації, приєднуємо до порту
        sleep 5
        ln -s /sys/kernel/config/nvmet/subsystems/ora20 /sys/kernel/config/nvmet/ports/1/subsystems/ora20
        ln -s /sys/kernel/config/nvmet/subsystems/ora20 /sys/kernel/config/nvmet/ports/2/subsystems/ora20
exit 0  
```
```
[root@pve99 ~]$ ./root/nvme_start.sh
[root@pve99 ~]$ apt install tree

[root@pve99 ~]$ tree /sys/kernel/config/nvmet/
/sys/kernel/config/nvmet/
├── hosts
├── ports
│   ├── 1
│   │   ├── addr_adrfam
│   │   ├── addr_traddr
│   │   ├── addr_treq
│   │   ├── addr_trsvcid
│   │   ├── addr_trtype
│   │   ├── addr_tsas
│   │   ├── ana_groups
│   │   │   └── 1
│   │   │       └── ana_state
│   │   ├── param_inline_data_size
│   │   ├── param_pi_enable
│   │   ├── referrals
│   │   └── subsystems
│   │       └── ora20 -> ../../../../nvmet/subsystems/ora20
│   └── 2
│       ├── addr_adrfam
│       ├── addr_traddr
│       ├── addr_treq
│       ├── addr_trsvcid
│       ├── addr_trtype
│       ├── addr_tsas
│       ├── ana_groups
│       │   └── 1
│       │       └── ana_state
│       ├── param_inline_data_size
│       ├── param_pi_enable
│       ├── referrals
│       └── subsystems
│           └── ora20 -> ../../../../nvmet/subsystems/ora20
└── subsystems
    └── ora20
        ├── allowed_hosts
        ├── attr_allow_any_host
        ├── attr_cntlid_max
        ├── attr_cntlid_min
        ├── attr_firmware
        ├── attr_ieee_oui
        ├── attr_model
        ├── attr_pi_enable
        ├── attr_qid_max
        ├── attr_serial
        ├── attr_version
        ├── namespaces
        │   └── 20
        │       ├── ana_grpid
        │       ├── buffered_io
        │       ├── device_nguid
        │       ├── device_path
        │       ├── device_uuid
        │       ├── enable
        │       ├── p2pmem
        │       └── revalidate_size
        └── passthru
            ├── admin_timeout
            ├── clear_ids
            ├── device_path
            ├── enable
            └── io_timeout

21 directories, 41 files
[root@pve99 ~]$   
```
## 2. A network is established between these nodes.

In this case, two node ports are connected directly.  

## 3. On one of the nodes we create a virtual machine “VSAN105” (10.10.1.3) based on Linux with the following characteristics (512Mb, 3Gb) and install the packages: “mdadm” (we will build a RAID1 array from the corresponding sections of the nodes’ nvme disks: nvme disc. , connect…, and then support it), “tgt” (for translating the md0 iscsi target to two nodes), “lvm2” (from the resulting md0 array we do pvcreate, vgcreate).  

### 3.1 Preparing a virtual machine for use as a cluster storage provider.  

### 3.2. Let's select Debian as an operating system.  

**We install :**  
```
apt install tgt

nano /etc/tgt/conf.d/tgtpve.conf  
```
**Filling:**  
```
<target iqn.1993-08.org.debian:01:9e746ebec3e>
   backing-store /dev/md0
   initiator-address 10.10.1.1
   initiator-address 10.10.1.2
</target>  
```
### 3.3. Installing "lvm2":  
```
apt install lvm2  
```
### 3.4. Install "nvme-cli":  
```
apt install nvme-cli  
```
**Loading modules at system startup:**  
```
modprobe nvme_tcp && echo "nvme_tcp" > /etc/modules-load.d/nvme_tcp.conf  
```
**Connect:**  
```
nvme discover -t tcp -a 10.10.1.1 -s 4420
nvme connect -t tcp -n ora10 -a 10.10.1.1 -s 4420
nvme discover -t tcp -a 10.10.1.2 -s 4420
nvme connect -t tcp -n ora20 -a 10.10.1.2 -s 4420  
```
**To connect at startup:**  
```
nano /etc/nvme/discovery.conf  
```
**Let's insert:**  
```
# Used for extracting default parameters for discovery
#
# Example:
# --transport=<trtype> --traddr=<traddr> --trsvcid=<trsvcid> --host-traddr=<host-traddr> --host-iface=<host-iface>
discover -t tcp -a 10.10.1.1 -s 4420
discover -t tcp -a 10.10.1.2 -s 4420  
```
**Then:**  
```
systemctl enable nvmf-autoconnect.service  
```
### 3.5. Preparing disks (changing sector size if necessary)  

**Noda 1**  
```
root@pve1:~# nvme list
Node Generic SN Model Namespace Usage Format FW Rev
/dev/nvme1n1 /dev/ng1n1 S4EUNG0M328258D Samsung SSD 970 EVO Plus 250GB 1 214.99 GB / 250.06 GB 512 B + 0 B 1B2QEXM7
/dev/nvme0n1 /dev/ng0n1 50026B7282A726A4 KINGSTON SKC3000S512G 1 512.11 GB / 512.11 GB 4 KiB + 0 B EIFK31.6  
```
**Checking the block size**  
```
root@pve1:~# nvme id-ns /dev/nvme0 -n 1 -H | grep &quot;LBA Format&quot;
[6:5] : 0 Most significant 2 bits of Current LBA Format Selected
[3:0] : 0x1 Least significant 4 bits of Current LBA Format Selected
LBA Format 0 : Metadata Size: 0 bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good
LBA Format 1 : Metadata Size: 0 bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better (in use)
root@pve1:~#  
```
**If necessary, change to 4k**  
```
root@pve1:~# nvme id-ns /dev/format --lbaf=1 /dev/nvme0n1  
```
### 3.6. Marking the disk.  
```
root@pve1:~# fdisk /dev/nvme0n1  
```
**We save the markings to use when we replace the disk and now for the duplicate:**  
```
root@pve1:~# sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512.dump
root@pve1:~# cat nvmeKINGSTON512.dump
label: gpt
label-id: A1F37274-73E6-864F-B0B6-9BDD551BBD45
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096
/dev/nvme0n1p1 : start= 4096, size= 262144, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=B21C1B97-64EE-6948-AA0F-0BBA8797EB91
/dev/nvme0n1p2 : start= 266240, size= 104857600, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=E71CF74F-B553-5246-A649-3A5C45619225
/dev/nvme0n1p3 : start= 105123840, size= 16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=47A78E4D-FC8E-F54B-8DAF-D52A7908590C
/dev/nvme0n1p4 : start= 121901056, size= 3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=7F23E207-9E1A-1B42-962F-98BED3C1F479  
```
**To restore this template later, you can do:**  
```
# sfdisk /dev/nvme0n1 < nvmeKINGSTON512.dump
root@pve1:~#  
```
**We split the second disk in the same way:**  
```
[root@pve99 ~]$ sfdisk /dev/nvme0n1 < nvmeKINGSTON512.dump
[root@pve99 ~]$ sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512_pve99.dump
[root@pve99 ~]$ cat nvmeKINGSTON512_pve99.dump
label: gpt
label-id: BB0E5F37-05C1-104A-8CB5-6D4A7DA9073A
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096
/dev/nvme0n1p1 : start= 4096, size= 262144, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=B139BB2A-C95D-1641-968F-3ED977E40B6A
/dev/nvme0n1p2 : start= 266240, size= 104857600, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=3CF0D8B4-A0C3-1145-B923-05D59A56B406
/dev/nvme0n1p3 : start= 105123840, size= 16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=35726BB3-0E33-C440-B5A1-14F31EE527FD
/dev/nvme0n1p4 : start= 121901056, size= 3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=4512150A-2074-5D44-90E3-58826E7A6152
[root@pve99 ~]$  
```
**Next we move on to the virtual machine:**  

### 3.7. Install mdadm:  
```
apt install mdadm  
```
### 3.8. Configuring Raid1 devices /dev/nvme1n1p2 , /dev/nvme2n1p2:  
```
nvme-list  
```
```
root@debvsan:/home/vov# nvme list  
```
|**Node**       |**Generic**|**SN**              |  Model	 | Namespace Usage | **Format**	         |FW	      |Rev      |  
|:-------------:|:---------:|:------------------:|:-------------:|:---------------:|:-------------------:|:----------:|:-------:|  
|/dev/nvme2n1	|/dev/ng2n1 |9782709cba71d57d8e6b|pve99-KINGSTON |20	           |512.11 GB / 512.11 GB| 4 KiB + 0 B| 6.8.12-5|  
|/dev/nvme1n1	|/dev/ng1n1 |f1f5c6929bb211d228f4|pve1-KINGSTON  |10	           |512.11 GB / 512.11 GB| 4 KiB + 0 B| 6.8.12-5|  

```
root@debvsan:/home/vov#
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme1n1p2 /dev/nvme2n1p2  
```
**Let's configure mdadm to reassemble the array during reboot, and then update the initrd to allow mdadm to stay there.**  
```
mdadm --detail --scan | tee -a /etc/mdadm/mdadm.conf
update-initramfs –u  
```
## 4. Configure "open-iscsi" on the nodes.  

### 4.1. Install packages on each node:  
```
apt install open-iscsi multipath-tools  
```
**Let's launch the service:**  
```
systemctl start open-iscsi.service  
```
### 4.2. On the first node we connect to the virtual machine, we find the target and log in:  
```
iscsiadm -m discovery -t st -p 10.10.1.3  
```
### 4.3. On the second node:  
```
[root@pve99:~iscsiadm -m discovery -t st -p 10.10.1.3
10.10.1.3:3260,1 iqn.1993-08.org.debian:01:9e746ebec3e  
```
## 5. On the virtual machine we do:  
```
# pvcreate /dev/md0
# vgcreate vg_nvme1 /dev/md0  
```
**On nodes and in the virtual machine, the command: vgs gives the display of the group: vg_nvme1.**  

## 6. In the Datacenter Storage LVM graphical shell, check the Shared box and select the nodes accordingly.  

## 7. We make a virtual machine to work only in RAM without a disk. We put it in HA. The system will be operational in case of failure of both nodes, both disks, or several network channels.  

### 7.1. In the file /usr/share/initramfs-tools/scripts/local, look for lines 179-185 (after making a backup of the file):  
```
   checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	if ! mount ${roflag} ${FSTYPE:+-t "${FSTYPE}"} ${ROOTFLAGS} "${ROOT}" "${rootmnt?}"; then
		panic "Failed to mount ${ROOT} as root file system."
	fi  
```
**And we change this code to the following:**  
```
   #checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	mkdir /ramboottmp
	mount ${roflag} -t ${FSTYPE} ${ROOTFLAGS} ${ROOT} /ramboottmp
	mount -t tmpfs -o size=100% none ${rootmnt}
	cd ${rootmnt}
	cp -rfa /ramboottmp/* ${rootmnt}
	umount /ramboottmp  
```
7.2. Save the file. And enter the command in the terminal as root:

mkinitramfs -o /boot/initrd.img-ramboot

7.3. Check that the file is created in the /boot folder and return the old local to its place in the /usr/share/initramfs-tools/scripts/local folder (or delete all our changes that we made in step 1).
7.4. Go to the folder: /etc and find the file: fstab, save a copy of it and edit it, look for something like this in the first lines:

UUID= 321dba83-9a22-442b-b06b-185d7afe1088 / ext4 defaults 1 1

and change to:

none / tmpfs defaults 0 0

7.5 Let's make the corresponding menu upon boot:

file:

nano /etc/grub.d/40_custom

Content:

#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'RAM-Debian GNU/Linux' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-321dba83-9a22-442b-b06b-185d7afe1088' {
        load_video
        insmod gzio
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        echo    'Loading Linux 6.1.0-28-amd64 ...'
        linux   /boot/vmlinuz-6.1.0-28-amd64 root=UUID=321dba83-9a22-442b-b06b-185d7afe1088 ro  quiet splash toram
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-ramboot
}

update-grub

We get the grub menu.
7.5. Now let's make ram.tar.gz, turn off the virtual machine, download a new virtual machine in liveCD mode, connecting the disk of this virtual machine.

Let's mount it to /mnt. Run:

# cd /mnt
# tar -czf /mnt/boot/ram.tar.gz .

7.6. Now when loading the virtual machine, select the appropriate boot menu in RAM. After loading, disconnect the disk:
7.6.1. Stop access to the disk:

To do this, use the command:

echo 1 > /sys/block/sda/device/delete`

This will disable the /dev/sda device at the kernel level. The disk will no longer be visible to the system.

7.6.2. Let's check the status:

Make sure the drive does not appear in the list of devices:

lsblk

It will look something like this:

root@debvsan:/home/vov# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                               8:0    0     8G  0 disk  
└─sda1                            8:1    0     8G  0 part  
root@debvsan:/home/vov#

echo 1 > /sys/block/sda/device/delete

lsblk

7.6.3. Disconnecting a disk from a virtual machine via QEMU monitor:
7.6.3.1 Enter the QEMU monitor for a specific VM:

qm monitor 105

Check all devices connected to the VM:

info block  

This will show all the connected drives. You will see something like this:

root@pve1:~# qm monitor 105 
Entering QEMU Monitor for VM 105 - type 'help' for help
qm> info block
drive-scsi0 (#block190): /dev/pve/vm-105-disk-1 (raw)
    Attached to:      scsi0
    Cache mode:       writeback, direct
    Detect zeroes:    unmap
qm>

Detach the disk:

device_del scsi0

Check what devices are connected:

info pci

You will see a list of PCI devices, including the SCSI controller.

For example :

Bus  9, device   1, function 0:
    SCSI controller: PCI device 1af4:1004
      PCI subsystem 1af4:0008
      IRQ 10, pin A
      BAR0: I/O at 0x1000 [0x103f].
      BAR1: 32 bit memory at 0xfd800000 [0xfd800fff].
      BAR4: 64 bit prefetchable memory at 0xfc000000 [0xfc003fff].
      id "virtioscsi0"

Delete:

qm> device_del virtioscsi0

Logout:

q

7.3.4.2. Insert into the configuration file:

root@pve1:~# nano /etc/pve/qemu-server/105.conf

Next: disabled=1

Example:

scsi0: local-lvm:vm-105-disk-1,disabled=1,aio=native,backup=0,discard=on,iothread=1,size=8G
scsihw: virtio-scsi-single,disabled=1

7.3.4.3. And during migration we will see:

()
Task viewer: VM 105 - Migrate
OutputStatus
Stop
Download
task started by HA resource agent
2025-01-04 00:34:24 use dedicated network address for sending migration traffic (10.10.1.1)
2025-01-04 00:34:24 starting migration of VM 105 to node 'pve1' (10.10.1.1)
2025-01-04 00:34:24 starting VM 105 on remote node 'pve1'
2025-01-04 00:34:28 start remote tunnel
2025-01-04 00:34:29 ssh tunnel ver 1
2025-01-04 00:34:29 starting online/live migration on unix:/run/qemu-server/105.migrate
2025-01-04 00:34:29 set migration capabilities
2025-01-04 00:34:29 migration downtime limit: 100 ms
2025-01-04 00:34:29 migration cachesize: 512.0 MiB
2025-01-04 00:34:29 set migration parameters
2025-01-04 00:34:29 start migrate command to unix:/run/qemu-server/105.migrate
2025-01-04 00:34:30 migration active, transferred 357.9 MiB of 4.0 GiB VM-state, 586.6 MiB/s
2025-01-04 00:34:31 migration active, transferred 735.9 MiB of 4.0 GiB VM-state, 543.8 MiB/s
2025-01-04 00:34:32 migration active, transferred 1.1 GiB of 4.0 GiB VM-state, 399.1 MiB/s
2025-01-04 00:34:33 migration active, transferred 1.3 GiB of 4.0 GiB VM-state, 502.5 MiB/s
2025-01-04 00:34:34 migration active, transferred 1.6 GiB of 4.0 GiB VM-state, 243.0 MiB/s
2025-01-04 00:34:35 migration active, transferred 1.8 GiB of 4.0 GiB VM-state, 320.5 MiB/s
2025-01-04 00:34:36 migration active, transferred 2.2 GiB of 4.0 GiB VM-state, 462.0 MiB/s
2025-01-04 00:34:37 migration active, transferred 2.5 GiB of 4.0 GiB VM-state, 443.7 MiB/s
2025-01-04 00:34:38 average migration speed: 457.0 MiB/s - downtime 73 ms
2025-01-04 00:34:38 migration status: completed
2025-01-04 00:34:42 migration finished successfully (duration 00:00:18)
TASK OK

8. We transfer the virtual machine in Dtacenter to HA.

Using the Proxmox graphical interface, we transfer vm 105 to HA.
9. Notification, diagnostics, and troubleshooting of RAID1 problems are the responsibility of the utilities in the “mdadm” package.
