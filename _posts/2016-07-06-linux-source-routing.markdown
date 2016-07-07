---
layout: post
title:  "Automatic Linux source based routing"
date:   2016-07-06 18:00:49 -0500
categories: ubuntu debian network routing
---

## The Problem

On a Linux server with multiple network interfaces configured on different subnets,
you will often find that the interface(s) which do not have the default route pointing
toward them will not handle incoming connections properly due to the asymmetrical
return path.

#### Example:
{% highlight bash %}
$ ip addr
...
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:fc:45:9d brd ff:ff:ff:ff:ff:ff
    inet 162.253.43.134/24 brd 162.253.43.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefc:459d/64 scope link
       valid_lft forever preferred_lft forever
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:2b:e7:89 brd ff:ff:ff:ff:ff:ff
    inet 10.13.96.161/19 brd 10.13.127.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe2b:e789/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}

{% highlight bash %}
$ ip route
default via 162.253.43.1 dev ens3
10.0.0.0/8 via 10.13.96.1 dev ens4
10.13.96.0/19 dev ens4  proto kernel  scope link  src 10.13.96.161
162.253.42.0/24 dev ens3  scope link
162.253.43.0/24 dev ens3  proto kernel  scope link  src 162.253.43.134
169.254.169.254 via 162.253.43.1 dev ens3
{% endhighlight %}

From my remote workstation I can ping 162.253.43.134 just fine.
{% highlight bash %}
$ ping -c1 162.253.43.134
PING 162.253.43.134 (162.253.43.134): 56 data bytes
64 bytes from 162.253.43.134: icmp_seq=0 ttl=60 time=5.825 ms
{% endhighlight %}

However, I can't access 10.13.96.161.
{% highlight bash %}
$ ping 10.13.96.161
PING 10.13.96.161 (10.13.96.161): 56 data bytes
Request timeout for icmp_seq 0
{% endhighlight %}

Why not? Let's check the incoming packet.
{% highlight bash %}
$ sudo tcpdump -n -i ens4 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens4, link-type EN10MB (Ethernet), capture size 262144 bytes
00:30:40.011105 IP 64.31.0.51 > 10.13.96.161: ICMP echo request, id 56759, seq 0, length 64
{% endhighlight %}

The packet comes in to the private interface, but since the system has `net.ipv4.conf.all.rp_filter`
enabled by default, the packet is simply dropped since the outgoing path (`ens3`, public,
where the default gateway points) is not the receiving interface.

## The Solution

To solve this problem of asymmetrical routing, we need to add a source-based
routing rule to the system so it will route all return traffic sourced from the
`ens4` private subnet `10.13.96.161/19` back out the correct interface.

First, create a routing table for your secondary interface
{% highlight bash %}
$ sudo echo '500     ens4' >> /etc/iproute2/rt_tables
{% endhighlight %}

Then drop the following script in `/opt/if-post-up-source-route`:
{% highlight bash %}
#!/bin/sh -e
#this script requires a routing table named $IFACE (ie. bond0) exists in /etc/iproute2/rt_tables
#the $IFACE routing table is used to place the default route in for the source routing table
#ip route list table $IFACE will list the routing table for this interface

set_netinfo() {
	IPADDR=$(ifconfig "$IFACE" | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}')
	NETMASK=$(ifconfig "$IFACE" | grep 'inet addr:' | cut -d: -f4 | awk '{ print $1}')

	OLDIFS=$IFS
	IFS=.
	set -- $IPADDR
	local IPADDR1=$1
	local IPADDR2=$2
	local IPADDR3=$3
	local IPADDR4=$4

	set -- $NETMASK
	local NETMASK1=$1
	local NETMASK2=$2
	local NETMASK3=$3
	local NETMASK4=$4
	IFS=$OLDIFS
	NETWORK=$(printf "%d.%d.%d.%d\n" "$((IPADDR1 & NETMASK1))" "$((IPADDR2 & NETMASK2))" "$((IPADDR3 & NETMASK3))" "$((IPADDR4 & NETMASK4))")
	GATEWAY=$(printf "%d.%d.%d.%d\n" "$((IPADDR1 & NETMASK1))" "$((IPADDR2 & NETMASK2))" "$((IPADDR3 & NETMASK3))" "$(((IPADDR4 & NETMASK4)+1))")
}

set_netinfo
ip route flush table "$IFACE"
ip route add "$NETWORK/$NETMASK" dev "$IFACE" proto kernel scope link table "$IFACE"
ip route add default via "$GATEWAY" dev "$IFACE" table "$IFACE"
ip rule del lookup "$IFACE" || true
ip rule add from "$NETWORK/$NETMASK" lookup "$IFACE"
{% endhighlight %}

Make the script executable:
{% highlight bash %}
$ sudo chmod +x /opt/if-post-up-source-route
{% endhighlight %}

Then edit your `/etc/network/interfaces` file containing the `ens4` configuration.
Add a `post-up /opt/if-post-up-source-route` line to the interface configuration.
Mine looks like:
{% highlight code %}
auto ens4
iface ens4 inet dhcp
  post-up /opt/if-post-up-source-route
{% endhighlight %}

Restart the interface:

__As always when restarting network interfaces, make sure you have a working out of band management method such as IPMI in case the interface fails to restart__
{% highlight bash %}
$ sudo ifdown ens4 && sudo ifup ens4
{% endhighlight %}

## Test the result
Check the `ens4` routing table:
{% highlight bash %}
$ ip route list table ens4
default via 10.13.96.1 dev ens4
10.13.96.0/19 dev ens4  proto kernel  scope link
{% endhighlight %}

Check the `ip rule` output for the ens4 source-based rule:
{% highlight bash %}
$ ip rule
0:	from all lookup local
32765:	from 10.13.96.0/19 lookup ens4
32766:	from all lookup main
32767:	from all lookup default
{% endhighlight %}

Test the source route:
{% highlight bash %}
$ ping -c1 10.13.96.161
PING 10.13.96.161 (10.13.96.161): 56 data bytes
64 bytes from 10.13.96.161: icmp_seq=0 ttl=60 time=5.511 ms
{% endhighlight %}

{% highlight bash %}
$ sudo tcpdump -n -i ens4 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens4, link-type EN10MB (Ethernet), capture size 262144 bytes
00:47:31.496909 IP 64.31.0.51 > 10.13.96.161: ICMP echo request, id 26040, seq 0, length 64
00:47:31.496969 IP 10.13.96.161 > 64.31.0.51: ICMP echo reply, id 26040, seq 0, length 64
{% endhighlight %}

This configuration is persistent across reboots. Simply repeat the steps above
if you have multiple interfaces that require source routing.
