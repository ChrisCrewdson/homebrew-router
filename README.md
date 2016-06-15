# homebrew-router
Setup instructions for homebrew router running ubuntu server 16.04.  Distilled from [here](http://arstechnica.com/gadgets/2016/04/the-ars-guide-to-building-a-linux-router-from-scratch/)


## Setup interface config
Determine interface names of your ethernet ports.  Use `ifconfig` to assist.  Annotate `/etc/network/interfaces` file with comments for future reference.  Configure static IP and subnet for LAN interface.  Replace `${WAN}` and `${LAN}` with actual interface names.

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The WAN interface
auto ${WAN}
iface ${WAN} inet dhcp

# The LAN interface
auto ${LAN}
iface ${LAN} inet static
        address 192.168.1.1
        netmask 255.255.255.0
```


## Enable packet forwarding between interfaces 
In `/etc/sysctl.conf`, add or uncomment the following 

```
net.ipv4.ip_forward=1
```

Run `sudo sysctl -p` to force changes to take effect immediately


## Load iptables rules on startup
Create this `/etc/network/if-pre-up.d/iptables` so router loads iptables rules before interfaces come online during boot sequence.  Add the following contents.

```
#!/bin/sh
/sbin/iptables-restore < /etc/network/iptables
```

Run the following to let the system execute it during boot sequence.

```
sudo chown root /etc/network/if-pre-up.d/iptables ; chmod 755 /etc/network/if-pre-up.d/iptables
```




## Configure iptables rules
Add the following to `/etc/network/iptables`.  Replace `${WAN}` and `${LAN}` with actual interface names.

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

#${WAN} is WAN interface, ${LAN} is LAN interface
-A POSTROUTING -o ${WAN} -j MASQUERADE

COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Service rules

#basic global accept rules - ICMP, loopback, traceroute, established all accepted
-A INPUT -s 127.0.0.0/8 -d 127.0.0.0/8 -i lo -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -m state --state ESTABLISHED -j ACCEPT

# enable traceroute rejections to get sent out
-A INPUT -p udp -m udp --dport 33434:33523 -j REJECT --reject-with icmp-port-unreachable

# DNS - accept from LAN
-A INPUT -i ${LAN} -p tcp --dport 53 -j ACCEPT
-A INPUT -i ${LAN} -p udp --dport 53 -j ACCEPT

# SSH - accept from LAN
-A INPUT -i ${LAN} -p tcp --dport 22 -j ACCEPT

# DHCP client requests - accept from LAN
-A INPUT -i ${LAN} -p udp --dport 67:68 -j ACCEPT

# drop all other inbound traffic
-A INPUT -j DROP

#Forwarding rules

#forward packets along established/related connections
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

#forward from LAN (${LAN}) to WAN (${WAN})
-A FORWARD -i ${LAN} -o ${WAN} -j ACCEPT

#drop all other forwarded traffic
-A FORWARD -j DROP

COMMIT
```

## Install DHCP Server
Run `sudo apt-get install isc-dhcp-server` to install DHCP.  


## Configure DHCP Server
Insert the following clause in `/etc/dhcp/dhcpd.conf`

```
subnet 192.168.1.0 netmask 255.255.255.0 {
        range 192.168.1.100 192.168.1.199;
        option routers 192.168.1.1;
        option domain-name-servers 192.168.1.1;
        option broadcast-address 192.168.1.255;
}
```

## Install DNS Server
Run `sudo apt-get install bind9`

## Change DNS servers for WAN interface
Add the following to `/etc/dhcp/dhclient.conf`

```
supersede domain-name-servers 8.8.8.8, 8.8.4.4;
```

## Restart router
Run `sudo shutdown -r now` to restart and make sure all changes take effect

## View DHCP leases
Examine contents of `/var/lib/dhcp/dhcpd.leases` to verify router is handing out IP addresses.  Use `dhcp_leases.py` in this repo to pretty-printify those contents