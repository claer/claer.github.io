---
layout: post
title:  "Mount Gluster NFS share on CoreOS"
date:   2016-12-12 23:48:31 +0100
categories: gluster nfs coreos
---
Previously this year I removed glusterfs from the debian nodes because of a problem upgrading them and I wasn't actually using the NFS shares. In this post, I'll try to reimplement it and to mounbt the gluster share as NFS from the coreos nodes.

## Reinstall Gluster
I followed the previous instructions I've written to reinstall Gluster FS in its 3.8 version.
It makes me realize I've forgotten to enable some network flows. So here is the relevant part for iptables modifications :

- for db-sc1
{% highlight conf %}
-A INPUT -i virbr1 -p udp -m multiport --dports 10053,111,2049,32769,875,892 -j ACCEPT
-A INPUT -i virbr1 -p tcp -m multiport --dports 10053,111,2049,32803,875,892 -j ACCEPT
-A INPUT -i virbr1 -p tcp -m tcp --dport 38465:38469 -j ACCEPT
{% endhighlight %}

- for db-xc1
{% highlight conf %}
-A INPUT -i br10 -p udp -m multiport --dports 10053,111,2049,32769,875,892 -j ACCEPT
-A INPUT -i br10 -p tcp -m multiport --dports 10053,111,2049,32803,875,892 -j ACCEPT
-A INPUT -i br10 -p tcp -m tcp --dport 38465:38469 -j ACCEPT
{% endhighlight %}

- Enable and start rpcstat daemon for nfs servers
{% highlight shell %}
[root@db-sc1 29 ~]# systemctl enable rpcbind.service
[root@db-sc1 29 ~]# systemctl start rpcbind.service
{% endhighlight %}

- Restart gluster service
{% highlight shell %}
[root@db-sc1 29 ~]# service glusterfs-server restart
{% endhighlight %}

- Here is the output you should get from ```gluster volume status```
{% highlight shell %}
[root@db-sc1 29 ~]# gluster volume status
Status of volume: nfsvol
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick db-xc1.claer.local:/data/nfs          49153     0          Y       9311
Brick db-sc1.claer.local:/data/nfs          49153     0          Y       14466
NFS Server on localhost                     2049      0          Y       1531
Self-heal Daemon on localhost               N/A       N/A        Y       1539
NFS Server on db-xc1.claer.local            2049      0          Y       10576
Self-heal Daemon on db-xc1.claer.local      N/A       N/A        Y       10584

Task Status of Volume nfsvol
------------------------------------------------------------------------------
There are no active volume tasks

[root@db-sc1 29 ~]# 
{% endhighlight %}

## CoreOS Integration
- Test mount a dummu volume
{% highlight shell %}
coreos1 ~ # systemctl start rpc-statd.service
coreos1 ~ # mount -t nfs -o vers=4,nfsvers=4 172.16.4.1:/nfsvol /mnt/nfs
{% endhighlight %}

After checking the mount is successfull and that you can write to it, modify the cloud init to make the changes persistent.

- Add a system unit on cloud_init file

{% highlight conf %}
    - name: data.mount
      command: start
      content: |
        [Unit]
        Before=docker.service
        [Install]
        RequiredBy=docker.service
        [Mount]
        What=172.16.4.1:/nfsvol
        Where=/data
        Type=nfs
{% endhighlight %}

References for shared storage on CoreOS:
* [Persistent volume in OpenShift and Kubernetes using Gluster](https://blog.gluster.org/2016/03/persistent-volume-and-claim-in-openshift-and-kubernetes-using-glusterfs-volume-plugin/)
