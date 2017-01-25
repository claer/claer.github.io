---
layout: post
title:  "Replace Libvirt routed network for OpenvSwitch"
date:   2016-09-19 23:30:30 +0100
categories: debian kvm libvirt openvswitch
---
I don'k know really why, out of the 2 servers I rent, one works like a charm (the slower one of course) the other one has terrible throughput when I download a file from the server to workstations in my LAN. After installing iperf and running various tests, I deducted the slow downs affected only VMs. When I was testing flows between physical hosts I had the correct results. Even when the communication is tunnelled with GRE over IPSEC.
For the fun, I wanted to try if the problem was the routed network on Libvirt, as I know it's not a common setup maybe it was the cause of my issues.

Following are the instructions I used to replace the routed network with one managed by OpenvSwitch.

## Installing OpenvSwitch
- Install the packages
{% highlight shell %}
[root@db-xc1 696 ~]# apt install openvswitch-switch
{% endhighlight %}

## Define the new network using Libvirt
- Delete the old network
{% highlight shell %}
[root@db-xc1 696 ~]# virsh net-destroy VMNetwork
{% endhighlight %}

- Create the xml entry for the new network
{% highlight xml %}
<network>
  <name>OVSNetwork</name>
  <forward mode='route'/>
  <bridge name='br10' stp='on' delay='0'/>
  <ip address='172.16.4.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='172.16.4.100' end='172.16.4.199'/>
    </dhcp>
  </ip>
  <virtualport type='openvswitch'/>
</network>
{% endhighlight %}

- Define the new network and start it automatically at boot
{% highlight shell %}
[root@db-xc1 696 ~]# virsh net-define /root/ovsnetwork.xml
[root@db-xc1 696 ~]# virsh net-autostart OVSNetwork
[root@db-xc1 696 ~]# virsh net-start OVSNetwork
{% endhighlight %}

## Update the Bird configuration file to adapt the routing
- Update on the /etc/bird/bird.conf file :
{% highlight conf %}
protocol device {
        scan time 60;
        primary 172.16.4.0/24;
}

protocol direct {
        interface "br10";
}
{% endhighlight %}

- Restart the bird daemon and check
{% highlight shelll %}
[root@db-xc1 35 ~]# service bird restart
{% endhighlight %}

