---
layout: post
title:  "GlusterFS on both Debian nodes, Cloud-init for CoreOS on Libvirt"
date:   2016-05-27 23:30:30 +0100
categories: debian glusterfs unbound coreos libvirt
---

With only 2 nodes, I cannot use ceph to share storage between nodes for container persistent storage. Gluster could be used with only 2 nodes and fulfil my requirements. My aim being to use docker/rkt on top of coreOS, I looked for gluster support on CoreOS. Unfortunately, CoreOS Devs don't want to add support for Gluster fs tools on CoreOS base. (more exaclty, I cannot use gluster client to mount gluster storage for docker usage). My only solution left is to use GlusteFS between my 2 Debian nodes then to export the clustered filesystem using NFS.

Today I also ameliored the DNS configuration in unbound and installed CoreOS using libvirt/KVM and cloud-init.

## Installing gluster
- Install gluster is straightforward, just follow the link high-availability-storage-with-glusterfs-on-debian-8-with-two-nodes referenced at the bottom of the page

- Creation of Gluster NFS volume
{% highlight shell %}
[root@db-xc1 675 ~]# gluster volume create nfsvol replica 2 transport tcp db-xc1.claer.local:/data/nfs db-sc1.claer.local:/data/nfs force
[root@db-xc1 675 ~]# gluster volume start  nfsvol
{% endhighlight %}

- On nodes, specify the socket binding address. Add the following line to the file /etc/glusterfs/glusterd.vol
{% highlight conf %}
option transport.socket.bind-address 172.16.4.1
{% endhighlight %}

- Don't forget to restart the daemon after editing the volume configuration
{% highlight shell %}
[root@db-xc1 675 ~]# service glusterfs-server restart
{% endhighlight %}

## Correcting DNS reverse zone
I forgot to declare the reverse zone for the GRE subnet. So I added the zone 255.16.172.in-addr.arpa to NSD & modified Unbound to route requests to the authoritative server.

## Deploying CoreOS with libvirt
I followed the CoreOS documentation to Install CoreOS with libvirt. To do so, you will need a recent libvirt with support for cloud-init config method. It is known that CentOS6/RHEL6 does not support disk based cloud-inits as proposed by CoreOS.

- Create the directory and store CoreOS image into it
{% highlight shell %}
[root@db-xc1 ~]# mkdir -p /var/lib/libvirt/images/coreos
[root@db-xc1 ~]# cd /var/lib/libvirt/images/coreos
[root@db-xc1 ~]# wget https://alpha.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2 -O - | bzcat > coreos_production_qemu_image.img
{% endhighlight %}

- Make a snapshot of the current image
{% highlight shell %}
[root@db-xc1 ~]# qemu-img create -f qcow2 -b coreos_production_qemu_image.img coreos1.qcow2
{% endhighlight %}

- Create the directory for the config drive
{% highlight shell %}
[root@db-xc1 ~]# mkdir -p /var/lib/libvirt/images/coreos/coreos1/openstack/latest
{% endhighlight %}

- Create the cloud-init file /var/lib/libvirt/images/coreos/coreos1/openstack/latest/user_data
{% highlight conf %}
# include one or more SSH public keys
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2E...==

hostname: "coreos1"

coreos:
  units:
    - name: etcd2.service
      command: start
    - name:  systemd-networkd.service
      command: restart
    - name: 10-eth0.network
      content: |
        [Match]
        Name=eth0

        [Network]
        Address=172.16.4.10/24
        Gateway=172.16.4.1
        DNS=172.16.4.1

  etcd2:
    name: coreos1
    advertise-client-urls: http://172.16.4.10:2379,http://172.16.4.10:4001
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    listen-peer-urls: http://172.16.4.10:2380
    initial-cluster-token: etcd-cluster-1
    initial-cluster: coreos1=http://172.16.4.10:2380
    initial-advertise-peer-urls: http://172.16.4.10:2380
    initial-cluster-state: new

  update:
    reboot-strategy: "best-effort"
{% endhighlight %}

- Create the VM with virt-install
{% highlight shell %}
[root@db-xc1 ~] virt-install --connect qemu:///system --import --name coreos1 --ram 1024 --vcpus 1 --os-type=linux --os-variant=virtio26 --disk path=/var/lib/libvirt/images/coreos/coreos1.qcow2,format=qcow2,bus=virtio --filesystem /var/lib/libvirt/images/coreos/coreos1/,config-2,type=mount,mode=squash --network network=VMNetwork --graphics spice --noautoconsole
{% endhighlight %}


## References for Gluster installation:

* [high-availability-storage-with-glusterfs-on-debian-8-with-two-nodes](https://www.howtoforge.com/tutorial/high-availability-storage-with-glusterfs-on-debian-8-with-two-nodes/)

