## Topology Diagrams 

> ![Network Topology](/images/topology.PNG)


## Multi site deployment featuring Network security

The project will involve 3 components or domains of my knowledge that is : Operating systems, Cloud and mainly  Networking. 
Just to mention I will not be using any cloud provider in this project but, more cloud related architectures in Data centers and terms that are learned in cloud. 

The sole purpose of this project is to configure and create a multi site deployment environment using real Cisco images hosted on the Eve-ng machine and where the networking emulator is hosted on.
The project has 2 sites, Site A and Site B. 

## Configured Features  

For site A i want to implement a Redundant and fault tolerant network design with multiple Switches and just a single Router.
Both networking infrastructure devices I got from Cisco CML which you do have to pay to obtain the license.
And we want to configure the following  : 

 - Configure hostname, Subnet Inital plan,
 - VLAN configuration that are divided into multiple Departments ( like HR, IT, DEV).
 - VLAN redunant trunking line to establish commnication from the same VLAN's 
 - Implemented EitherChannel Portchannel 0 configured with redundancy and Security. 
 - Added Inter Vlan Routing mechanism with Router on a Stick method.
 - ROAS has multiple sub interfaces for each VLAN departmant
 - Configured Dynamic NAT protcol with ACL rules
 - Connctivity between the Sites is eastablished via Static Routing

This ends and concludes the Site A configration, This is not a real representation of how firms actually work or some corporate network. Much more functinallity, automation, network programmability could be added.
This is from my view as a CCNA student on how would i implment a Network design.
For the Site B, i didnt really configure anything special, the Site B could act as fail over if a disaster stucts all the devices are connected just needs to be configured.

### VLAN Configuration 

To logically segment the network and from one Network i divided it into subnets, so for this 
Vlans were created based on departments. 
This allows devices in th same VLAN to communicate even if they are on the different Switch.
In this conifiguration i used 
- VLAN 10 IT Department subnet is 10.10.0.0/24 -> (VPC1, VPC3)
- VLAN 20 HR Department subnet is 10.20.0.0/24 -> (VPC2)
- VLAN 30 DEV Department subnet is 10.30.0.0/24 -> (VPC4, VPC5)

```bash
Switch>en
Switch# conf terminal
Switch(config)# hostname SW1 // Configure the name for the switch
SW1(config)# vlan 10 // Adding vlans command 
SW1(config-vlan)# name IT // naming them is optional
SW1(config-vlan)# exit
SW1(config)# interface gigabitEthernet 1/0
SW1(config-if)# switchport mode access  
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```

### VLAN trunking 
To be able to send data from one Switch to another between them we need some sort of trunking.
Trunking allows us to carry traffic to All VLAN's (except the Native one which i configured)
That is done via a term called `Tagging`,
So If we have VPC1 on SW1 and VPC3 on SW2, the line between the switches needs to be configured so that VPC1 can ONLY talk to hosts in his own VLAN. (In our case VPC3)

```bash
SW1(config)#interface range gigabitEthernet 0/0-2 // I used mutliple interfaces 
SW1(config-if-range)# switchport trunk encapsulation dot1q 
SW1(config-if-range)# switchport mode trunk 
SW1(config-if-range)# switchport trunk allowed vlan add 10
SW1(config-if-range)# switchport trunk allowed vlan add 20
SW1(config-if-range)# switchport trunk allowed vlan add 30
```

### Inter VLAN Routing 
In a VLAN bsaed network, devices in different VLANs are logically seperated.
If they are physically connected to the same switch.
This means that VPCS in VLAN 10 cannot directly communicate with other VPCS in other VLANS withpout a Layer 3 device.
The idea of Inter VLAN routing solves this issue or using a method called ROAS.
ROAS allows us to use sub interfaces instead of using each cable for each VLAN.

```bash
Router> enable
Router# configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)# hostname R1
R1(config)# interface gigabitEthernet 0/1 
R1(config-if)# no shutdown
R1(config-if)# interface gig0/1.10 //Repeat for each vlan the last 3 commands
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 10.10.0.1 255.255.255.0
R1(config-subif)# exit
```
![R1 Interface config](/images/R1Interfaces.PNG)

### Either channel 
To improve Redundancy and fault tolerance over our network, we need to configure Either channel.
This technology allows us to use muliple interfaces and act as one, this way if one fails due to some kind of disater, others continue to work. (This is called aggregation)
For Either channel to work it is needed to configure LACP protocol
Both sites need to be either in Active/Active or Active/Passive

```bash
SW1(config)# interface range gig0/0-2
SW1(config-if-range)# channel-group 1 mode active
SW1(config)#do show ip interface brief 
Port-channel1   unassigned   YES unset    up    up   
```

### Dyanmic NAT + ACL
So lets start with NAT, this is a protocol that mainly helps us translate the private address from each VLAN's subnet to a public IP address.
Dynamic NAT simulates accesss to the internet and lets us ping hosts to Site B.
So Dynamic NAT lets us define a set or pool of public addresses, i used only /30
An a host on VLAN 10, with a private address get translated in a complex process to a public ad 
![R1 Public address](/images/public.PNG)

With NAT, in addidtion i used Access Control list or ACL, to implement some kind of security and define who has access to outside network, who can ping outside nodes.
ACL isnt all about Controlling access, its also about reshaping the network achitecture based on our liking and what is actually needed.
For ACL's i only specified the 2 private subnets that are allowed to be translated.
Other subnets that are not defined will be not cos there is an DENY ALL at the end of ACE.

#### Here is the configuration for both NAT and ACL

```bash
R1>en
R1# conf te
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)# interface gigabitEthernet 0/1
R1(config-subif)# ip nat inside
R1(config-subif)# end
R1# configure terminal
R1(config)# interface gigabitEthernet 0/0
R1(config-subif)# ip nat outside
R1(config-subif)# end
R1> enable
R1# configure terminal
R1(config)# ip acess-list standard uki-NAT-ACL
R1(config-std-nacl)# permit 10.10.0.0 0.0.0.255
R1(config-std-nacl)# permit 10.30.0.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# ip nat pool Our-NAT-Pool 25.10.5.3 25.10.5.6 255.255.255.248
R1(config)# ip nat inside source list uki-NAT-ACL pool Our-NAT-Pool
```

#### The picture below is a command that displays the utilization of NAT pool
[R1 NAT Pool](/images/NAT-conf.PNG)

