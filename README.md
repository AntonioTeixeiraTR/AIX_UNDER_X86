# AIX_UNDER_X86
Steps to create an AIX vm emulated under Qemu Linux Virtual Machine

## 1 - you will need an AIX iso  ( a 7.2 TL 5 would do the trick )

### instfix -i | grep ML
    All filesets for 7.2.0.0_AIX_ML were found.
    All filesets for 7200-00_AIX_ML were found.
    All filesets for 7200-01_AIX_ML were found.
    All filesets for 7200-02_AIX_ML were found.
    All filesets for 7200-03_AIX_ML were found.
    All filesets for 7200-04_AIX_ML were found.
    All filesets for 7200-05_AIX_ML were found.

## 2 - OS version and QEMU

### I am using fedora 41 alpha and qemu version 8   ( that combination worked  well with AIX 7.2  the 7.1 not so much )

  	root@ryzen9:/mnt# rpm -qa | grep -i qemu-ppc
  	root@ryzen9:/mnt# rpm -qa | grep -i qemu | grep -i ppc
  	qemu-user-static-ppc-8.2.2-1.fc41.x86_64  

### root@host:/dir# neofetch
```
             .',;::::;,'.                root@ryzen9
         .';:cccccccccccc:;,.            -----------
      .;cccccccccccccccccccccc;.         OS: Fedora Linux 41 (KDE Plasma Prerelease) x86_64
    .:cccccccccccccccccccccccccc:.       Kernel: 6.9.0-0.rc0.20240322git8e938e398669.14.fc41.x86_64
  .;ccccccccccccc;.:dddl:.;ccccccc;.     Uptime: 2 days, 8 hours, 59 mins
 .:ccccccccccccc;OWMKOOXMWd;ccccccc:.    Packages: 3737 (rpm), 10 (flatpak)
.:ccccccccccccc;KMMc;cc;xMMc:ccccccc:.   Shell: bash 5.2.26
,cccccccccccccc;MMM.;cc;;WW::cccccccc,   Resolution: 3440x1440
:cccccccccccccc;MMM.;cccccccccccccccc:   Terminal: /dev/pts/3
:ccccccc;oxOOOo;MMM0OOk.;cccccccccccc:   CPU: AMD Ryzen 9 5950X (32) @ 3.400GHz
cccccc:0MMKxdd:;MMMkddc.;cccccccccccc;   GPU: AMD ATI Radeon RX 470/480/570/570X/580/580X/590
ccccc:XM0';cccc;MMM.;cccccccccccccccc'   Memory: 20959MiB / 63692MiB
ccccc;MMo;ccccc;MMW.;ccccccccccccccc;
ccccc;0MNc.ccc.xMMd:ccccccccccccccc;
cccccc;dNMWXXXWM0::cccccccccccccc:,
cccccccc;.:odl:.;cccccccccccccc:,.
:cccccccccccccccccccccccccccc:'.
.:cccccccccccccccccccccc:;,..
  '::cccccccccccccc::;,.

root@host:/dir#
```

## 3 -  Create a file to be the DISK of your AIX system
```
root@host:/dir#qemu-img create -f  qcow2  hdisk0.qcow2  20G
```
## 4 - CREATE the AIX VM using the qemu binary
### Remember to change cpu, memory and file names to the ones you specified
```
root@host:/dir#qemu-system-ppc64 -cpu POWER9 -smp 4 \  
-M pseries,ic-mode=xics -m 16384 -serial stdio \
-drive file=hdisk0.qcow2,if=none,id=drive-virtio-disk0 \   
-device virtio-scsi-pci,id=scsi \
-device scsi-hd,drive=drive-virtio-disk0 \
-cdrom AIX_Install_7.2.5.6_DVD_1.iso \     
-prom-env "boot-command=boot cdrom:" \
-prom-env "input-device=/vdevice/vty@71000000" \
-prom-env "output-device=/vdevice/vty@71000000"
```
### At this point we are creating it without network configuration and booting from cdrom because we will need to tweak it to boot from disk later.

## 4.1 Follow the Steps and Menus to perform an instalation of the system 

### Be sure to choose to install SSH in the Menu options to avoid the need to do it later
### Never do a Ctrl+C cause it will make your system to crash.
### Be absolutly patience - even if you have tons of resources this will be an emulated system, therefore it will take time to complete actions.

## 5 - You can create a Linux interface to have access through SSH

### Create TAP interface
```
root@host:/dir#ip tuntap add dev tap0 mode tap
```
### Enable proxy_arp on both devices (the tap device and your LAN/WLAN interface)
```
root@host:/dir#echo 1 > /proc/sys/net/ipv4/conf/tap0/proxy_arp

root@host:/dir#echo 1 > /proc/sys/net/ipv4/conf/wlp6s0f1u4/proxy_arp   
```

### Configure the IP accordingly with your network
```
root@host:/dir#ip addr add 192.168.100.12 dev tap0

root@host:/dir#ip link set up tap0

root@host:/dir#ip link set up dev tap0 promisc on

root@host:/dir#ip route add 192.168.100.200 dev tap0

root@host:/dir#arp -Ds 192.168.100.200 wlp6s0f1u4 pub  
```
### We will not be covering Linux tap configurations over here, if you wanna get deeper  chatGPT and Google is your friend  :)

## 6 - Booting from the ISO and fixing your AIX

### At this point you need to boot from the cdrom again and choose the Maintenance mode and mount the root volume
### Once that is done, you need to erase a file and change for the lines below :

### Go to /sbin/helpers/jfs2 and empty the file fsck64 :
```
root@host:/dir#cd /sbin/helpers/jfs2
root@host:/dir#> fsck64
```
### Edit the fsck64 file with the following lines:
```
#!/bin/ksh

exit 0
```
### Save and quit the file.
```
root@host:/dir#sync;sync

root@host:/dir#halt
```
## 7 -  Boot from the disk and finish the configuration

### You will have to choose console, languages, terminal and set root password once that is done you will have an AIX system ready to do any necessary test.
### Here is the final Qemu line so you can boot from the disk :
```
qemu-system-ppc64 -cpu POWER9 -smp 4 \
-M pseries,ic-mode=xics -m 16384 -serial stdio \
-drive file=hdisk0.qcow2,if=none,id=drive-virtio-disk0 \
-device virtio-scsi-pci,id=scsi \
-device scsi-hd,drive=drive-virtio-disk0 \
-cdrom AIX_Install_7.2.5.6_DVD_1.iso \
-net nic -net tap,script=no,ifname=tap0
-prom-env "boot-command=boot disk:" \
-prom-env "input-device=/vdevice/vty@71000000" \
-prom-env "output-device=/vdevice/vty@71000000"
```
## 8 - Post installation instructions to avoid problems
```
rmitab cron
rmitab clcomd
rmitab naudio2
rmitab pfcdaemon
stopsrc -s clcomd
stopsrc -s pfcdaemon
```
### Network configuration so you can ssh from your host to AIX and avoid the risk to Ctrl+C by accident
```	
chdev -l en0 -a netaddr=192.168.100.200 -a netmask=255.255.255.0 -a state=up   <-------- again, remember to change ip addresses to yours ip 
```
### Ps.: you can make this AIX access the internet if you configure your Linux network  ( tap and external NIC ) to be on a bridge, but this is out of the scope of this guide
### Ps.2:  the ps command could crash dump your system, so remember to use TOPAS  - not all binaries work well under the qemu.



  
