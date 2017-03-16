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

There is some resources available on this subject already, but they either doesn't work, or doesn't cover the whole process of setting it up. With this article I'm trying to change that. So if there's anything I've missed or forgotten to add, please [create an issue](https://github.com/Lochnair/blog/issues/new) at the blog's repository.


## Assumptions ##
- You're using an EdgeRouter as the client
- You're running Ubuntu Server (15.04 or later) on the endpoint
- You don't need OpenVPN's security functions
- You have 1 IP address on the server per connection (either IPv4 or IPv6)


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

You should also change the MTU to what fits your network. Find the MTU for your uplink(s) using [this guide](http://www.dslreports.com/faq/695), and subtract 46 from the result when using IPv4, 66 when using IPv6, this accounts for the overhead of OpenVPN. If the MTU for your uplinks differ, use the lowest value.

```
auto bond0
iface bond0 inet static
  address 10.75.0.1
  netmask 255.255.255.0
  mtu 1434
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

```
set interfaces bonding bond0 address 10.75.0.2/24
set interfaces bonding bond0 description WAN_BOND
set interfaces bonding bond0 mode round-robin
```

## Credits ##
I would like to mention [this question](http://stackoverflow.com/questions/10698886/openvpn-bond-2-tap-tunnels) over at StackOverflow. The poster answered his own question, and it was a great help when first setting this up.