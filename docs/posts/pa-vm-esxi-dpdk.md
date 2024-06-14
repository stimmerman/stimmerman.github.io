---
authors: [stimmerman]
comments: true
categories:
  - Palo Alto
  - VMware
date: 2024-06-14
tags:
  - palo alto
  - dpdk
  - vmware
---
# PA-VM on ESXi 'interface configured but down'
When deploying a PA-VM (11.1.3) on ESXI (8.0U2) I ran into a problem where the management network was working, but all other interfaces showed up as: **interface configured but down**. 

TLDR: `set system setting dpdk-pkt-io off`

<!-- more -->

## Troubleshooting

I checked:

- The mapping of the NICS as configured on the VM at the ESXI host and inside the PA-VM.
- Configured the interfaces from 'auto' to 'on' at the PA-VM.
- The NICS showed as up on the port groups on ESXI, the statistics even showed some packets.

However, the interfaces stayed down in the PA-VM...

Issueing `show interface all` on the CLI showed the interfaces as `ukn/ukn/down((autoneg))`:

```
admin@xxxx(active)> show interface all

total configured hardware interfaces: 3

name         id    speed/duplex/state       mac address       
----------------------------------------------------------------
ethernet1/1  16    ukn/ukn/down((autoneg))  00:50:56:97:28:83 
ethernet1/2  17    ukn/ukn/down((autoneg))  00:50:56:97:64:df 
ethernet1/4  19    ukn/ukn/down((autoneg))  00:50:56:97:d8:25 

aggregation groups: 0

total configured logical interfaces: 3

name         id    vsys zone   forwarding  tag  address  
----------------- ------------------------- ----------------
ethernet1/1  16    1           ha          0    10.158.255.0/31   
ethernet1/2  17    1           ha          0    10.158.255.2/31   
ethernet1/4  19    1           N/A         0    N/A               
```

Using `debug show vm-series interfaces all` showed all interfaces as assigned to the PA-VM. I noticed the difference between the driver being used for the `mgt` (working) and `Ethernetx/x` (broken) interfaces:
```
admin@xxxx(active)> debug show vm-series interfaces all 

Interface_name  Base-OS_port  Base-OS_MAC        PCI-ID        Driver
 mgt                eth0     00:50:56:97:d4:4c  0000:03:00.0     vmxnet3
 Ethernet1/1        eth1     00:50:56:97:28:83  0000:0b:00.0  net_vmxnet3
 Ethernet1/2        eth2     00:50:56:97:64:df  0000:13:00.0  net_vmxnet3
 Ethernet1/3        eth3     00:50:56:97:7e:25  0000:1b:00.0  net_vmxnet3
 Ethernet1/4        eth4     00:50:56:97:d8:25  0000:04:00.0  net_vmxnet3
 Ethernet1/5        eth5     00:50:56:97:c9:ca  0000:0c:00.0  net_vmxnet3
 Ethernet1/6        eth6     00:50:56:97:04:c7  0000:14:00.0  net_vmxnet3
 Ethernet1/7        eth7     00:50:56:97:e6:2f  0000:1c:00.0  net_vmxnet3
 Ethernet1/8        eth8     00:50:56:97:75:0b  0000:05:00.0  net_vmxnet3
 Ethernet1/9        eth9     00:50:56:97:72:9b  0000:0d:00.0  net_vmxnet3
```

## PacketMMAP and DPDK

After a while I started checking the compatibility matrixes, and stumbled on the [PacketMMAP and DPDK Drivers on VM-Series Firewalls](https://docs.paloaltonetworks.com/compatibility-matrix/vm-series-firewalls/sr-iov-and-dpdk-drivers) page. 

>By default DPDK is enabled on VM-Series firewalls as stated below. If the VM-Series firewall detects an unsupported driver, the firewall reverts to PacketMMap mode.

You can check the status of the packet IO mode on the PA-VM with: `show system setting dpdk-pkt-io`

```
admin@xxxx> show system setting dpdk-pkt-io

Device current Packet IO mode:                 DPDK
Device DPDK Packet IO capable:                 yes
Device default Packet IO mode:                 DPDK
```

## Solution

In my case DPDK was not supported, but the PA-VM somehow did not revert to PacketMMap mode. Using `set system setting dpdk-pkt-io off` followed by a reboot forced the PA-VM to use PacketMMap mode:

```
admin@xxxx(active)> set system setting dpdk-pkt-io off
Warning: Enabling/disabling DPDK Packet IO mode will reboot the device. Do you want to continue? (y or n) 

Broadcast message from root (console) (Fri Jun 14 01:16:12 2024):

The system is going down for reboot NOW!

Device is now in non-DPDK IO mode, rebooting device
```

And tadaa, my interfaces were up and finally working:

```
admin@nlcm1m1-fwl001> show interface all

total configured hardware interfaces: 4

name         id  speed/duplex/state  mac address
-------------------------------------------------------
ethernet1/1  16  10000/full/up       00:50:56:97:28:83 
ethernet1/2  17  10000/full/up       00:50:56:97:64:df 
ethernet1/3  18  10000/full/up       00:50:56:97:7e:25 
ethernet1/4  19  10000/full/up       00:50:56:97:d8:25 

aggregation groups: 0

total configured logical interfaces: 7

name             id    vsys zone  forwarding   tag    address
--------------------- ------------------------- ------------------
ethernet1/1      16    1          ha           0      10.158.255.0/31
ethernet1/2      17    1          ha           0      10.158.255.2/31
ethernet1/3      18    1          N/A          0      N/A            
ethernet1/4      19    1          N/A          0      N/A            
ethernet1/4.103  258   1          vr:default   103    N/A            
ethernet1/4.105  256   1          vr:default   105    N/A            
ethernet1/4.106  257   1          vr:default   106    10.158.33.1/24 
```