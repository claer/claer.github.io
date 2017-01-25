---
layout: post
title:  "Add Unbound on Debian hosts"
date:   2016-05-20 23:30:30 +0100
categories: debian unbound
---
After reinstalling the home firewall, I wanted to add some dns cache to local domain on each rented servers. For this usage, I installed unbound on Debian and ordered it to redirect local queries to my home dns server.

## Unbound local cache and forwarder
- First, let's install the package
{% highlight shell %}
[root@db-sc1 308 ~]# apt install unbound
{% endhighlight %}

- Configuration of Unbound.

Welcome to Debian... Debian likes to break configuration on small files, even when the rest of the world is not doing so. To comply to Debian file structure, we will modify the following files:
 - /etc/unbound/unbound.conf.d/localbind.conf
 - /etc/unbound/unbound.conf.d/remotecontrol.conf
 - /etc/unbound/unbound.conf.d/stubzone.conf

The file localbind.conf will control the bind address and the access controls. The file remotecontrol.conf is used to access remote control from localhost. The file stubzone.conf will describe all local zones that needs to be redirected.

- Local bind: /etc/unbound/unbound.conf.d/localbind.conf
{% highlight config %}
server:
        interface: ::1
        interface: 127.0.0.1
        access-control: 0.0.0.0/0 refuse
        access-control: 127.0.0.0/8 allow
        access-control: ::0/0 refuse
        access-control: ::1 allow
        access-control: ::ffff:127.0.0.1 allow

        hide-identity: yes
{% endhighlight %}

- Remote control: /etc/unbound/unbound.conf.d/remotecontrol.conf
{% highlight config %}
remote-control:
        control-enable: yes
        control-interface: 127.0.0.1
        control-interface: ::1
{% endhighlight %}

- Stub zone definition: /etc/unbound/unbound.conf.d/stubzone.conf

{% highlight conf %}
stub-zone:
        name: "claer.local"
        stub-host: "172.16.2.1"
stub-zone:
        name: "1.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "2.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "3.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "4.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "5.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "6.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
stub-zone:
        name: "7.16.172.in-addr.arpa"
        stub-host: 172.16.2.1
{% endhighlight %}

- Reload configuration
{% highlight shell %}
[root@db-xc1 682 ~]# service unbound restart
{% endhighlight %}
