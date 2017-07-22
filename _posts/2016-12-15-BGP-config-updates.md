---
layout: post
title:  "Updating BGP configuration"
date:   2016-12-15 23:48:31 +0100
categories: bird bgp openbsd coreos calico
---
Up to now, my BGP topology consisted of 3 AS, one per physical machine (and location). If I wish to have a "floating" subnet between the 2 nodes in datacenter, they need to share the same AS. That's why I decided to move from eBGP to iBGP between all the nodes (calico and physical).
Calico was configured in iBGP by default, that means I should have a complete full mesh of all nodes. Not really a problem with my previous tolology, however now it is best to setup route reflection between the 2 physical nodes.
In order to have a nice topology, once route reflectors are setup on each physical machine I should disable full mesh peering in Calico and agents should be peering route reflectors only.

## Bird
- The bird daemon should be modified accordingly to take into account the new topology. Here is the config diff.

{% highlight shell %}
[root@db-xc1 511 ~]# diff -u /etc/bird/bird.conf.orig /etc/bird/bird.conf
--- /etc/bird/bird.conf.orig    2016-12-16 10:22:00.923775558 +0100
+++ /etc/bird/bird.conf 2016-12-16 18:01:29.917886143 +0100
@@ -32,6 +32,8 @@

 protocol direct {
        interface "br10";
+       interface "gre1";
+       interface "gre2";
 }

 function net_infra() {
@@ -54,33 +56,32 @@
         else reject;
 }

-# Small peer
+# Peering between servers
 protocol bgp db_sc1 {
         local as myas;
         description "to db-sc1";
-        neighbor 172.16.255.9 as 65003;
+        neighbor 172.16.255.9 as myas;
         import filter bgp_in;
         import limit 20;
         export filter bgp_out;
+       next hop self;
 }
 }
-
-protocol bgp coreos1 {
+protocol bgp cobra {
         local as myas;
-        description "to coreos1";
-        neighbor 172.16.4.10 as myas;
-       next hop self;
+        description "to home";
+        neighbor 172.16.255.5 as 65001;
+       source address 172.16.255.6;
         import filter bgp_in;
         import limit 20;
-        export filter {
-               if net_infra() then accept;
-               if net_martian() then reject;
-               else accept;
-       };
+        export filter bgp_out;
+       next hop self;
 }
-protocol bgp docker {
-        local as myas;
-        description "to docker";
-        neighbor 172.16.4.105 as myas;
+
+# template bgp for nodes docker/rkt
+template bgp rr_client {
+       local as myas;
+       multihop;
+       rr client;
        next hop self;
         import filter bgp_in;
         import limit 20;
@@ -91,11 +92,16 @@
        };
 }
-protocol bgp cobra {
-        local as myas;
-        description "to home";
-        neighbor 172.16.255.5 as 65001;
-        import filter bgp_in;
-        import limit 20;
-        export filter bgp_out;
+protocol bgp coreos1 from rr_client {
+        description "to coreos1";
+       neighbor 172.16.4.10 as myas;
 }
+protocol bgp coreos3 from rr_client {
+        description "to coreos3";
+       neighbor 172.16.4.11 as myas;
+}
+protocol bgp docker from rr_client {
+        description "to docker";
+        neighbor 172.16.4.105 as myas;
+}
+
[root@db-xc1 512 ~] 1 #
{% endhighlight %}

- Same thing has to be applied to the other node
{% highlight shell %}
[root@db-sc1 484 ~] 130 # diff -u /etc/bird/bird.conf.orig /etc/bird/bird.conf
--- /etc/bird/bird.conf.orig    2016-12-16 10:49:28.783677298 +0100
+++ /etc/bird/bird.conf 2016-12-16 18:01:36.701416272 +0100
@@ -9,7 +9,7 @@
 # of your router, usually one of router's IPv4 addresses.
 router id 172.16.3.1;

-define myas = 65003;
+define myas = 65004;

 # The Kernel protocol is not a real routing protocol. Instead of communicating
 # with other routers in the network, it performs synchronization of BIRD's
@@ -27,10 +27,13 @@
 # interfaces from the kernel.
 protocol device {
        scan time 60;
+       primary 172.16.3.0/24;
 }

 protocol direct {
         interface "virbr1";
+        interface "gre1";
+        interface "gre2";
 }

 function net_infra() {
@@ -52,13 +55,14 @@
        else reject;
 }

-# Small peer
+# Peering entre les serveurs
 protocol bgp db_xc1 {
        local as myas;
        description "to db-xc1";
-       neighbor 172.16.255.10 as 65004;
+       neighbor 172.16.255.10 as myas;
+       next hop self;
        import filter bgp_in;
-       import limit 10;
+       import limit 20;
        export filter bgp_out;
 }

@@ -66,7 +70,29 @@
        local as myas;
        description "to home";
        neighbor 172.16.255.1 as 65001;
+       source address 172.16.255.2;
        import filter bgp_in;
-       import limit 10;
+       import limit 20;
        export filter bgp_out;
 }
+
+# template bgp for nodes docker/rkt
+template bgp rr_client {
+        local as myas;
+        multihop;
+        rr client;
+        next hop self;
+        import filter bgp_in;
+        import limit 20;
+        export filter {
+                if net_infra() then accept;
+                if net_martian() then reject;
+                else accept;
+        };
+}
+
+protocol bgp coreos2 from rr_client {
+        description "to coreos2";
+        neighbor 172.16.3.10 as myas;
+}
+
[root@db-sc1 485 ~] 1 #

{% endhighlight %}

## BGPd

- Doing so, I have asymetrical routing on OpenBSD, as the daemon insert routes to always match 1 entry point for the global AS and does not have optimal routing. For a quick fix, I entered a local pref on OpenBSD.

{% highlight conf %}
# temp fix localpref for direct routing prefix 172.16.4/24
match from $peer2 prefix 172.16.4.0/24 set {localpref 200}
{% endhighlight %}

## Calico

- Disabling full mesh
{% highlight shell %}
root@ubuntu-docker:~# calicoctl config set nodeToNodeMesh off
{% endhighlight %}

This is not finished, I should inform Calico that it has to establish a peering with the physical node:

- Deleting global bgp peer

{% highlight shell %}
coreos2 bin # ./calicoctl delete bgpPeer 172.16.4.1 --scope=global
{% endhighlight %}

- Configuring the 2 RR as peers
{% highlight shell %}
coreos2 bin # ./calicoctl create -f /tmp/bgppeer
Successfully created 1 'bgpPeer' resource(s)
coreos2 bin # cat /tmp/bgppeer
apiVersion: v1
kind: bgpPeer
metadata:
  peerIP: 172.16.3.1
  scope: node
  node: coreos2
spec:
  asNumber: 65004

coreos1 ~ # cat /tmp/bgppeer
apiVersion: v1
kind: bgpPeer
metadata:
  peerIP: 172.16.4.1
  scope: node
  node: coreos1
spec:
  asNumber: 65004

coreos1 ~ # /srv/bin/calicoctl create -f /tmp/bgppeer
Successfully created 1 'bgpPeer' resource(s)
coreos1 ~ # /srv/bin/calicoctl get bgppeer
SCOPE   PEERIP       NODE      ASN
node    172.16.4.1   coreos1   65004
node    172.16.3.1   coreos2   65004
{% endhighlight %}

## CoreOS

Small update on CoreOS, the etcd2 service should be set to "start" and not "enable" with systemd.
