# vagrant-hyperv

This repo describes how to setup a Hyper V guest via Vagrant.

## Prerequisites

Window 10 Pro with Hyper V enabled

## Hyper V setup

Since Hyper V on Win10 comes with the so called "Default" Network switch that changes his network range on every reboot, we create first a natted Network with an static ip range.

This network has no dhcp enabled so you need care of the ip assignment in the vm by yourself.

First create a new internal network switch:

```
New-VMSwitch -Name NATSwitch -SwitchType Intern
```

Get the interface index of the new switch:

```
Get-NetAdapter | Format-Table -AutoSize

Name                       InterfaceDescription                                 ifIndex Status MacAddress        LinkSpeed 
----                       --------------------                                 ------- ------ ----------        --------- 
vEthernet (Default Switch) Hyper-V Virtual Ethernet Adapter                          23 Up     E6-15-9B-23-5C-BC   10 Gbps 
vEthernet (NATSwitch)      Hyper-V Virtual Ethernet Adapter #2                       13 Up     00-15-5D-01-77-09   10 Gbps 
WiFi                       Killer Wireless-n/a/ac 1535 Wireless Network Adapter       8 Up     9C-B6-D0-16-80-89  400 Mbps

```

In this case it is 23.

Setup ip for the new switch/IF:

```
New-NetIPAddress -IPAddress 10.0.2.1 -PrefixLength 24 -InterfaceIndex 23
```

Create new Nat-Object on this IF:

```
New-NetNat -Name MyNAT -InternalIPInterfaceAddressPrefix 10.0.2.0/24
```

## VM setup

Just use the Vagrantfile in this repo and start the VM:

```
vagrant.exe up
```

During vagrant starting the VM you will be asked which Network switch to use, choose here the just created Nat switch.

## Post VM setup

After the VM is booted, connect to it via `vagrant.exe ssh` or the Hyper V console and setup a static IP for the VM

You can use the netplan file from this repo or set it up an other way you like.

If you plan to use the VM for eg a libvirt setup aka nested VM, disable mac address spoofing:

```
Set-VMNetworkAdapter -VMName <vm name> -MacAddressSpoofing off
```

## Speedup VM boot

Hyper V setup the VM in this way that it tries to boot first from UEFI, Network etc.

I find this slow and decided to change the boot order:

```
 Set-VMFirmware -VMname <vm name> -FirstBootDevice (Get-VMHardDiskDrive -VMName <vm name>)[0]
```

## Provide service from the vm to the network

If you want to make a service in the vm reachable from your network, we need to create a port forwarding to it:

```
Add-NetNatStaticMapping -NatName MyNAT -Protocol TCP -ExternalIPAddress 0.0.0.0 -ExternalPort 2222 -InternalIPAddress 10.0.2.49 -InternalPort 22
```

Here is create a port forwarding for the ssh servive in the vm with the ip 10.0.2.49, it will be reachable on port 2222 of the host.
