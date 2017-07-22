---
layout: post
title:  "Correcting errors on BGP Route Reflector configuration"
date:   2016-12-23 23:48:31 +0100
categories: bird bgp openbsd calico
---
This post is an update to my previous configuration. The problem with route reflectors is I had to set the next hop to self. Doing so, from the node cobra (that peer with both route reflectors) all prefixes are prefered from one route reflector only. it means we don't have optimal routing in place. To enhance the current topology, I decided to use AS prepend between the 2 route reflectors. That way, subnets will have automatically best path to destination from cobra.

## Bird
- Seting up the BGP out filter in bird to add AS prepend:

{% highlight conf %}
filter bgp_out {
        if ( from = 172.16.255.10 ) then bgp_path.prepend(65004);
        if net_martian() then reject;
        else accept;
}
{% endhighlight %}

- Here is the resulting RIB table on cobra

{% highlight shell %}
root@cobra ~ # bgpctl sh r
flags: * = Valid, > = Selected, I = via IBGP, A = Announced, S = Stale
origin: i = IGP, e = EGP, ? = Incomplete

flags destination          gateway          lpref   med aspath origin
AI*>  172.16.1.0/24        0.0.0.0            100     0 i
AI*>  172.16.2.0/24        0.0.0.0            100     0 i
*>    172.16.3.0/24        172.16.255.2       100     0 65004 i
*     172.16.3.0/24        172.16.255.6       100     0 65004 65004 i
*>    172.16.4.0/24        172.16.255.6       200     0 65004 i
*     172.16.4.0/24        172.16.255.2       100     0 65004 65004 i
*>    172.16.5.128/26      172.16.255.6       100     0 65004 i
*     172.16.5.128/26      172.16.255.2       100     0 65004 65004 i
*>    172.16.5.192/26      172.16.255.2       100     0 65004 i
*     172.16.5.192/26      172.16.255.6       100     0 65004 65004 i
*>    172.16.6.0/26        172.16.255.2       100     0 65004 i
*     172.16.6.0/26        172.16.255.6       100     0 65004 65004 i
*>    172.16.255.5/32      172.16.255.6       100     0 65004 i
*>    172.16.255.9/32      172.16.255.6       100     0 65004 i

{% endhighlight %}

## Calico

By default, with calico there is a default deny policy, to at least allow ICMP, here is the relevant configuration

{% highlight shell %}
core@coreos1 ~ $ cat pol.yml
- apiVersion: v1
  kind: policy
  metadata:
    name: global-icmp
  spec:
    ingress:
    - action: allow
      protocol: icmp
    egress:
    - action: allow
      protocol: icmp
core@coreos1 ~ $
coreos1 ~ # /srv/bin/calicoctl create -f ~core/pol.yml
{% endhighlight %}
