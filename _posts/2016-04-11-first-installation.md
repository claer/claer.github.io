---
layout: post
title:  "First Dedibox customisations"
date:   2016-04-11 23:48:31 +0100
categories: security kvm ipsec network
---
This blog aims to detail the day to day changes to my infrastructure. That one is composed of 2 rented servers and 1 PC Engines APU2 for OpenBSD firewall at home. The aim is to have redundant services (if possible) while testing new technologies. The 2 rented servers come with Debian Jessie preinstalled.

Today I configured ipsec + gre tunnels between the 3 boxes and created some routed LAN for kvm on the rented server.
I also hardened the server with iptables and added fail2ban.

## Fail2ban
- Install fail2ban package
{% highlight shell %}
root@db-sc1 28 ~]# apt install fail2ban
{% endhighlight %}

- In /etc/fail2ban/jail.conf modify the line ignoreip and increase ban time
{% highlight conf %}
ignoreip = 127.0.0.1/8 <openbsd_ip>
# one week ban time
bantime  = 604800
{% endhighlight %}

- Restart service
{% highlight shell %}
# systemctl restart fail2ban.service
{% endhighlight %}

## KVM
- Install packages and enable packet forwarding on the box
{% highlight shell %}
[root@db-sc1 29 ~]# apt install libvirt-bin libvirt-daemon libvirt-clients
[root@db-sc1 30 ~]# echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/10-forwarding.conf
{% endhighlight %}

- Create a routed network for VM
Note: Do not create the network in /etc/network/interfaces. Libvirtd is taking care of creating the interface virbr1 and affecting the IP address.

 - Create a file named vmnetwork.xml with the following content:
{% highlight xml %}
<network>
  <name>VMNetwork</name>
  <bridge name="virbr1"/>
  <forward mode='route'/>
  <ip address="172.16.3.1" netmask="255.255.255.0">
    <dhcp>
      <range start="172.16.3.100" end="172.16.3.199"/>
    </dhcp>
  </ip>
</network>
{% endhighlight %}

 - Enable the new network
{% highlight shell %}
[root@db-sc1 32 ~]# virsh net-define /root/vmnetwork.xml 
Network VMNetwork defined from /root/vmnetwork.xml

[root@db-sc1 34 ~]# virsh net-autostart VMNetwork
Network VMNetwork marked as autostarted

[root@db-sc1 35 ~]# virsh net-start VMNetwork 
Network VMNetwork started

[root@db-sc1 36 ~]# 
{% endhighlight %}

- Create the outgoing rule
{% highlight shell %}
[root@db-sc1 529 ~]# iptables -t nat -A POSTROUTING -s '172.16.3.0/24' -o eth0 -j MASQUERADE
{% endhighlight %}

- Make the iptables changes persistent by first, creating a file named /etc/network/if-pre-up.d/iptables

{% highlight shell %}
[root@db-sc1 529 ~]# cat /etc/iptables.up.rules
*nat
:PREROUTING ACCEPT [204:14725]
:INPUT ACCEPT [60:5991]
:OUTPUT ACCEPT [148:11892]
:POSTROUTING ACCEPT [148:11892]
-A POSTROUTING -s 172.16.3.0/24 -o eth0 -j MASQUERADE
COMMIT
# 
{% endhighlight %}

{% highlight shell %}
[root@db-sc1 528 ~]# cat /etc/network/if-pre-up.d/iptables
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.up.rules
[root@db-sc1 529 ~]# chmod a+x /etc/network/if-pre-up.d/iptables
{% endhighlight %}

## IPSEC and GRE
- Install packages
{% highlight shell %}
root@db-sc1 531 ~]# apt install strongswan
{% endhighlight %}
- Create the /etc/ipsec.conf file
{% highlight conf %}
conn openbsd-cobra
        left=<my_ip>
        leftsubnet=<my_ip>/32
        leftauth=psk
        right=<openbsd_ip>
        rightsubnet=<openbsd_ip>/32
        rightauth=psk
        type=tunnel
        auto=start
	keyexchange = ikev1
        ike=aes256-sha256-modp2048
        ikelifetime = 24h
        esp=aes256-sha256-modp2048
        lifetime = 24h
        dpddelay = 30s
        dpdaction = restart
        keyingtries = %forever
        authby=psk
        closeaction=restart
{% endhighlight %}

- Add your passphrase to /etc/ipsec.secrets
{% highlight conf %}
<openbsd_ip> %any : PSK "mypsk"
{% endhighlight %}
 
- Create the GRE tunnel interface by adding the file /etc/network/interfaces.d/gre1
{% highlight conf %}
auto gre1
iface gre1 inet static
    address 172.16.255.2
    netmask 255.255.255.252
    pre-up ip tunnel add gre1 mode gre local <my_ip> remote <openbsd_ip> ttl 255
    up ifconfig gre1 multicast
    pointopoint 172.16.255.1
    post-down iptunnel del gre1
{% endhighlight %}

- Reboot and verify with ipsec status


References for IPSEC:
* [install libreswan on debian](http://www.routerperformance.net/howtos/install-libreswan-on-debian-jessie/)
* [ipsec vpn tunnel on raspbian using strongswan](https://bourskov.dk/2016/01/ipsec-vpn-tunnel-on-raspbian-using-strongswan/)
