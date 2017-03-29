---
title: Bonding internet connections
s: bonding-internet-connections
date: 2017-03-13 15:49:04
tags:
- bonding
- dsl 
- internet
- edgerouter
---

## Background ##
I live in a rural area where the only means of getting online is 4,5/0,4 Mbit DSL. As you can imagine this becomes a big problem when multiple people are trying to use the line at the same time (read: [bufferbloat](http://www.dslreports.com/faq/17883)). Recently we got an extra line from another ISP in hope that we could improve the situation somewhat by combining them.

There are some resources available on this subject already, but they either doesn't work, or doesn't cover the whole process of setting it up. With this article I'm trying to change that. So if there's anything I've missed or forgotten to add, please [create an issue](https://github.com/Lochnair/blog/issues/new) at the blog's repository.


## Assumptions ##
- You're using an EdgeRouter as the client
- You're running a Debian-based server on the endpoint
- You don't need OpenVPN's security functions


## Setting up the server ##
First of all we've got to set up the server that'll act as our gateway to the internet. To accomplish this, we're going to set up two OpenVPN servers simulating two point-to-point ethernet connections.

### Bonding interface ###
Open the `/etc/modules` file with your editor of choice and add the `bonding` module to it. This'll make the `bonding` module load on system boot.

```
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
bonding
```

Now we need to create the file `/etc/modprobe.d/bonding.conf`. Here we'll set the options for the module. Full documentation on the `bonding` module is available [here](https://www.kernel.org/doc/Documentation/networking/bonding.txt).

```
# mode=0 means round-robin
# miimon=100 sets the time between link checks to 100ms
options bonding mode=0 miimon=100
```

Now we'll have the `bond0` interface available after booting the server, but it doesn't have an IP address yet. So we need to edit the `/etc/network/interfaces` file and add the following. Change `10.75.0.1` to whatever address you'd like, as with the netmask.

You should also change the MTU to what fits for your network. Find the MTU for your uplink(s) using [this guide](http://www.dslreports.com/faq/695), and subtract 46 from the result when using IPv4, 66 when using IPv6, this accounts for the overhead of using OpenVPN. If the MTU for your uplinks differ, use the lowest value.

I'm also adding a `post-up` command, that'll add a route for 192.168.0.0/24 (assuming this is your LAN subnet) on the `bond0` interface. This way we don't need to set up NAT on the EdgeRouter, and thus avoid double-NAT'ing.

```
auto bond0
iface bond0 inet static
  address 10.75.0.1
  netmask 255.255.255.0
  mtu 1434
  post-up ip route add 192.168.0.0/24 via 10.75.0.2
```

#### NAT ####
Since multiple clients are going to connect to the internet from the same public IP address, we need to set up NAT. First make sure the `iptables-persistent` package is installed, then add a NAT rule that modifies the source address on all packets going out the WAN interface to your public IP.

You should also configure the firewall to secure the server and your network, but that's outside the scope of this article.

```
apt install iptables-persistent
iptables -t nat -A POSTROUTING -o <WAN_INTERFACE> -j SNAT --to-source <PUBLIC_IP>
iptables-save > /etc/iptables/rules.v4
```

### OpenVPN ###
First we need to create the TAP devices. Open the `/etc/network/interfaces` file again and add the contents below. For the sake of clarification, we're setting the interface down right after it's creation because you can't add an active interface as a bond slave. Adding the device as a slave brings it up again, which we don't want to happen until the OpenVPN connection is established, so we're bringing it down again.

```
auto tap0
iface tap0 inet manual
  pre-up openvpn --dev-type tap --dev tap0 --mktun
  post-up ip link set tap0 down
  post-up echo "+tap0" > /sys/class/net/bond0/bonding/slaves
  post-up ip link set tap0 down

auto tap1
iface tap1 inet manual
  pre-up openvpn --dev-type tap --dev tap1 --mktun
  post-up ip link set tap1 down
  post-up echo "+tap1" > /sys/class/net/bond0/bonding/slaves
  post-up ip link set tap1 down
```

This is the server OpenVPN config we're using. We need one OpenVPN instance per internet connection, so create one file in `/etc/openvpn` per instance. For example `/etc/openvpn/isp_1.conf` and `/etc/openvpn/isp_2.conf`. You need to change the `local` option to the IP address you'd like OpenVPN to bind to, and the `dev` option to one of the TAP devices you created earlier.

```
# Encryption adds extra overhead, so I'm disabling it.
# If you need the extra security, set the auth method and cipher you like,
# and remove the no-iv option.
auth none
cipher none
no-iv
 
# Use a TAP device for layer 2 operation
# Bonding by its nature doesn't work on TUN devices (layer 3)
# Change this to one of your TAP devices
dev tap0
 
# Use peer-to-peer mode
mode p2p
 
# Change to the port you'd like
port 5116
 
# Set the IP address to listen on here
local 12.34.56.78
 
# Use UDP for best performance
# If you can avoid it, you should never use TCP with OpenVPN
# If you're planning to listen on IPv6, change this to udp6.
proto udp
 
# Enable more verbose logging
verb 3

# Don't close the TAP device on restart
# Instead we're keeping it open and setting it to down by script.
persist-tun
 
# Send a OpenVPN ping to the other side every second
ping 1

# If we don't receive a ping within 10 seconds, restart the instance
ping-restart 10
 
# Wait until connection is up before opening the TAP device
# and running the "up" script
up-delay

# Make sure up/down scripts runs if ping-restart are triggered
up-restart

# Run down script before TAP close
# This should make sure no packets are lost on a dead connection
down-pre
 
# Allow running external scripts
script-security 2
 
# Set down/up scripts
down /etc/openvpn/tap-down
up /etc/openvpn/tap-up
```

Now we need to create the down/up scripts for OpenVPN. These scripts simply set the TAP interface down or up, depending on whether the OpenVPN connection is up or not. This allows the bond interface to determine which interface is available for sending on.

Add this to `/etc/openvpn/tap-down`
```
#!/bin/bash
ip link set $dev down
```

Add this to `/etc/openvpn/tap-up`
```
#!/bin/bash
ip link set $dev up
```

Make sure the scripts are executable:

```
chmod +x /etc/openvpn/tap-down
chmod +x /etc/openvpn/tap-up
```

And last enable the OpenVPN service, so our servers are started on system boot:

```
systemctl enable openvpn.service
```

Restart the server to make our changes active.

## Setting up the client ##
As mentioned earlier, I'm using an EdgeRouter (EdgeRouter Lite to be specific) for this setup, so all configuration examples will only work on the EdgeRouter series of devices (and maybe Vyatta/VyOS).

### Bonding interface ###
First we need to create the bond interface, make sure that the subnet matches what you set on the server.

```
set interfaces bonding bond0 address 10.75.0.2/24
set interfaces bonding bond0 description WAN_BOND
set interfaces bonding bond0 mode round-robin
```

### OpenVPN ###

Now we need to set up the OpenVPN clients. Just to be clear, normally you would configure the client only by using `set interfaces openvpn` commands. However I've not figured out how to disable the security features using that, so we're doing this with `.ovpn` files instead.

It doesn't really matter where you save them, as long as it's beneath the /config folder. Like on the server, we need one file per connection.

```
# Disable encryption
auth none
cipher none
no-iv

# Use a TAP device for layer 2
dev-type tap
dev vtun10

# Use peer-to-peer mode
mode p2p

# Set this to the IP address of your WAN interface
local 10.20.30.40

# Set remote
remote 12.34.56.78 5116 udp

# Enable more logging
verb 3

# Preserve TAP device
persist-tun

# Make sure connection is up
ping 1
ping-restart 5

# Wait until connection is up before opening the TAP device
# and running the "up" script
up-delay

# Make sure up/down scripts runs if ping-restart are triggered
up-restart

# Run down script before TAP close
# This should make sure no packets are lost on a dead connection
down-pre

# Allow running external scripts
script-security 2

# Set down/up scripts
down /config/scripts/tap-down
up /config/scripts/tap-up
```
  
Create the `/config/scripts/tap-down` script bringing the TAP device down on connection close.

```
#!/bin/bash
ip link set $dev down
```

Create the `/config/scripts/tap-up` script bringing the TAP device up when the connection is established. You'll notice that this one does more than the one we created on the server. This is because EdgeOS doesn't support adding OpenVPN TAP interfaces to a bond. So as a workaround we're adding the TAP interface to the bond when the connection is established, IF it isn't already.

```
#!/bin/bash

if ! grep $dev /sys/class/net/bond0/bonding/slaves; then
  ip link set $dev down
  echo "+$dev" > /sys/class/net/bond0/bonding/slaves
fi

ip link set $dev up
```

```
set interfaces openvpn vtun10 config-file /config/vpn-isp1.ovpn
set interfaces openvpn vtun10 description "VPN_ISP1"
set interfaces openvpn vtun11 config-file /config/vpn-isp2.ovpn
set interfaces openvpn vtun11 description "VPN_ISP2"
```

### Routing ###

Now on to a tricky part. By default all VPN connections would go over one WAN interface, which we have to avoid. The trick is using multiple routing tables, and selecting which one to use based on the source IP address of the connection. I've added dummy networks in this example, so make sure to replace them with whatever gateway and netmask your ISPs are using.

```
set protocols static table 10 description "VPN from ISP1 uplink"
set protocols static table 10 interface-route 10.20.30.0/24 next-hop-interface eth1
set protocols static table 10 route 0.0.0.0/0 next-hop 10.20.30.1

set protocols static table 11 description "VPN from ISP2 uplink"
set protocols static table 11 interface-route 10.20.40.0/24 next-hop-interface eth2
set protocols static table 11 route 0.0.0.0/0 next-hop 10.20.40.1
```

If you haven't done so already, now would be a good time to commit your changes. And preferably save them too.

Last thing to do is adding the rules for selecting which routing table to use for which traffic.

```
#!/bin/bash
ip rule add pref 10 from 10.20.30.40 iif lo lookup 10
ip rule add pref 11 from 10.20.40.40 iif lo lookup 11
```

## Acknowledgments ##
I would like to mention [this question](http://stackoverflow.com/questions/10698886/openvpn-bond-2-tap-tunnels) over at StackOverflow. The poster answered his own question, and it was a great help when first setting this up.