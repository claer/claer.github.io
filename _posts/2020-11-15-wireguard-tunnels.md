---
layout: post
title:  "Upgrading the whole infrastructure"
date:   2020-11-15 13:30:30 +0100
categories: debian kvm libvirt openvswitch ipv6 wireguard openbsd bgp
---

There is a major upgrade to my network. I got fiber connection at home so I
took the time to change the IPSec stack I was not so happy with. My goal here
is to replace the GRE+IPSec tunnels with Wireguard. Wireguard is nice because
it is a true L3 only layer. The encapsulation footprint is quite small with an
inner MTU of 1420.

I'll post here from the moment I finished the local customization up to
networking success. In 4 years since the begining of this blog a lot has
changed in the software land with so much new options. Unfortunately, prices
have not changed for the better and I keep my current dediboxes as they offer
better price/performance compared to current offer.

## Design goal

The plan is to keep the triange between home and 2 dediboxes. On home side, I
also keep OpenBSD as VPN router. The main difference here, I have to think my
home connection as outgoing flows only. The OrangeBox has a crappy limited
interface when it comes to networking. The option to change the device with a
dedicated router and ONT is not an option here. I don't want to spend money on
router and 10G switches for home use.
Currently, the box is awful and is buggy enougth to not support incoming flows
for fixed IPs inside LAN. 

So, the prerequisites for the topology is as follow:
- Configuration should adapt to the CrappyLiveBox
- Subnets should be exchanged dynamically between each triange corner
- If one link goes down, the communication should continue to flow
- If I move one VM from one dedibox to the other, the modifications on host should be minimal
  - Implement Public DNS master/slave zone
  - Implement Internal DNS master/slave zone

## Selecting technology and topology

As said earlier, I seleted Wireguard against IPSec because IPSec is complex and
can be a pain to troubleshoot. Particuliarly, if you think I don't have public
IP any more and I have only outgoing flows available at the home edge.

With Wireguard, there are 2 main topologies to choose from. Either you build
all VPNs on a single WG interface, therefore, you let Wireguard select the
route to the destination by configuring dedicated prefixes per connection.
Either you create as much interfaces as connections, and you decide routing
ouside Wireguard.
Because of my requirement to exchange subnets dynamically, I choose the 2nd
option. Wireguard does not permit to select route based on availability and
only use advertized prefixes at the tunnel establishement.

So on each node, I'll have 2 interfaces, wg1 and wg2, corresponding to each
tunnel. This setup has the disadvantage to use 2 UDP ports per host instead of
1 on normal operation.

Another cavehat, Debian requires either Stable distribution with kernel
backported or Testing.
At the begining I went to keep Stable and installed Wireguard from backports.
Unfortunately, I ran into another issue with Libvirt that forced me to upgrade
to Testing. To upgrade from Stable to Testing, please see your usual Debian
source of information. 

For the routing daemon on the OpenBSD side, I decided to keep OpenBGPd. It
works (I never had a glitch) for my simple usecase and I was able to transcribe
my needs into configuration easily.
For the routing daemon on Debian, I had few choices: keep Bird or move to FRR.
FRR comes from Quagga, a daemon I had issues with in the past, and not a very
good feeling approach. FRR is certainly much better than Quagga by now, but I
still have my bad experience in head. So I decided for Bird. The question now
is what version of Bird I should aim for. I was happy with version 1.6, but all
new devs are done on 2.0.x branch. The new reinstall was a perfect time to play
with the new version. The drawback here, I have to compile bird from hand.
Neither Debian nor Bird provice packages ready for Debian (maybe yes, I just
didn't find them). Bird provides packages for Ubuntu, not really my cup of tea.


## Wireguard generate the encryption keys

The process is different on Debian and OpenBSD. The OpenBSD method requires you
to bring an interface up and destroy it afterwards. Therefore I'll show you
here how to generate keys for all hosts on Debian. You will find the way to get
keys from OpenBSD tooling while reading the blog post. 

- Key material for Debian 1
{% highlight shell %}
wg genkey | tee /etc/wireguard/privatekey_debian1.txt | wg pubkey | tee /etc/wireguard/publickey_debian1.txt
{% endhighlight %}
- Key material for Debian 2
{% highlight shell %}
wg genkey | tee /etc/wireguard/privatekey_debian2.txt | wg pubkey | tee /etc/wireguard/publickey_debian2.txt
{% endhighlight %}
- Key material for OpenBSD
{% highlight shell %}
wg genkey | tee /etc/wireguard/privatekey_openbsd.txt | wg pubkey | tee /etc/wireguard/publickey_openbsd.txt
{% endhighlight %}

## OpenBSD Wireguard configuration

The configuration of wireguard on OpenBSD is really easy. As always with
OpenBSD, read the man page, it helps!
Basically, I replaced the previous GRE interfaces with WG ones and adapted the
configuration with Wireguard specific options: encryption, tunneled IPs.
Here is the configuration of the 2 hostname.wg? interfaces:

The important point is to use a prefix with both endpoints on the inner
addresses. In my case, I went from /32 to /30 so that I have one IP for the
other side of the tunnel.

You'll see also that I moved from IPv4 to IPv6 for outer connection. As CrapBox
provides native IPv6 for internal LAN, why bother with NAT. At least it let me
experiment the IPv4 inside IPv6 option :)

For the IP ranges allowed inside the tunnel, I've been very generous by adding
the whole B and C classes private prefixes.

- Get IP from CrapLiveBox
{% highlight shell %}
echo "inet 192.168.1.2/24" >> /etc/hostname.em2
echo "inet6 autoconf" >> /etc/hostname.em2
sh /etc/netstart em2
{% endhighlight %}

- Generate secret key with openssl
{% highlight shell %}
openssl rand -base64 32
<PrivateKey of OpenBSD firewall>
{% endhighlight %}

- Create /etc/hostname.* files
file /etc/hostname.wg1

{% highlight conf %}
wgport 51820 wgkey <PrivateKey of OpenBSD firewall>
wgpeer <PublicKey of 1st Dedibox> wgendpoint fc80:fc80:fc80:300::1 51820 wgaip 192.168.0.0/16 wgaip 172.16.0.0/12 wgaip 10.0.0.0/8
inet 172.16.255.1/30 # private IP attached to the network interface
{% endhighlight %}

file /etc/hostname.wg2

{% highlight conf %}
wgport 51820 wgkey <PrivateKey of OpenBSD firewall>
wgpeer <PublicKey of 2nd Dedibox> wgendpoint fc80:fc80:fc80:400::1 51820 wgaip 192.168.0.0/16 wgaip 172.16.0.0/12 wgaip 10.0.0.0/8
inet 172.16.255.5/30 # private IP attached to the network interface
{% endhighlight %}

- Get the public keys for each interface
{% highlight shell %}
PUB1="`ifconfig wg1 | grep 'wgpubkey' | cut -d ' ' -f 2`"
PUB2="`ifconfig wg2 | grep 'wgpubkey' | cut -d ' ' -f 2`"
{% endhighlight %}

- Bring up the tunnel
{% highlight shell %}
sh /etc/netstart wg1 wg2
{% endhighlight %}

## Debian IPv6 configuration

As often with Debian, everything is simple and complicated at the same time.
Simple because things do work out of the box. Complicated because it's never
what you expect from standard package or does not match your needs.

First, getting IPv6 public IP is a challenge. Dibbler previously used is not
supported anymore by Online.net. The new recommendation is to use dhcp client.
I assume here, you get the Prefix Delegation ID (DUID) from the Web interface.

- Create the file /etc/dhcp/dhclient6.conf
{% highlight conf %}
interface "enp0s20f0" {
   send dhcp6.client-id 00:01:02:03:04:05:06:07:08:09;
}
{% endhighlight %}

- Configure RA on the egress interface to be able to forward and receive dhcp6 address
{% highlight conf %}
cat << _EOF >> /etc/sysctl.d/10-forwarding.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.default.accept_ra = 2
net.ipv6.conf.enp0s20.accept_ra = 2
net.ipv6.conf.all.forwarding = 1
_EOF
{% endhighlight %}

- Create the Systemd unit: /etc/systemd/system/dhclient.service
{% highlight conf %}
[Unit]
Description=dhclient for sending DUID IPv6
After=network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=10
Type=forking
ExecStart=/sbin/dhclient -cf /etc/dhcp/dhclient6.conf -6 -P -v enp0s20
ExecStop=/sbin/dhclient -x -pf /var/run/dhclient6.pid

[Install]
WantedBy=network.target
{% endhighlight %}

- Start and enable the daemon
{% highlight shell %}
systemctl start dhclient.service
systemctl enable dhclient.service
{% endhighlight %}

- Now we have IPv6 prefix, add an IP address to the egress interface by adding this piece of code to /etc/network/interface
{% highlight conf %}
iface enp0s20 inet6 static
    address fc80:fc80:fc80:300::1
    netmask 64
{% endhighlight %}

- Add the address by hand if you wish immediate connectivity
{% highlight shell %}
ip -6 add add fc80:fc80:fc80:300::1/64 dev enp0s20
{% endhighlight %}

## Debian Wireguard configuration

- Installing wireguard once on Testing was easy
{% highlight shell %}
apt install wireguard wireguard-tools
{% endhighlight %}

- Create the keys following tutorials:
{% highlight shell %}
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
{% endhighlight %}

- Configuration issues
Once the configuration is in place, I had an issue with my VPN tunnels not
working the way I expected. This is because by default Wireguard on linux
insert routes configured for allowed IPs in the main routing table. OpenBSD did
not have these defaults. Needless to say which way I prefer...

- Final configuration for Wireguard on Debian: /etc/wireguard/wg1.conf
{% highlight conf %}
[Interface]
Address = 172.16.255.2/30
SaveConfig = false
ListenPort = 51820
PrivateKey = <PrivateKey of the host>
Table = off

# Cobra
[Peer]
PublicKey = <PublicKey of OpenBSD firewall>
Endpoint = [fc08:fc08:fc08:fc08:fc08:fa1:df6c:13de]:51820
AllowedIPs = 172.16.0.0/12, 192.168.0.0/16, 10.0.0.0/8
{% endhighlight %}

- Use the same template for /etc/wireguard/wg2.conf file

- Bring up the tunnel
{% highlight shell %}
wg-quick up wg1
wg-quick up wg2
{% endhighlight %}

- Check status
{% highlight shell %}
root@db-sc1:~# wg
interface: wg1
  public key: <PublicKey of the host>
  private key: (hidden)
  listening port: 51820

peer: <PublicKey of OpenBSD firewall>
  endpoint: [fc08:fc08:fc08:fc08:fc08:fc08:1c48:84c]:51820
  allowed ips: 172.16.0.0/12, 192.168.0.0/16, 10.0.0.0/8
  latest handshake: 1 minute, 19 seconds ago
  transfer: 55.82 MiB received, 1.49 GiB sent

interface: wg2
  public key: <PublicKey of the host>
  private key: (hidden)
  listening port: 51821

peer: <PublicKey of the other Dedibox>
  endpoint: 163.0.0.1:51821
  allowed ips: 172.16.0.0/12, 192.168.0.0/16, 10.0.0.0/8
  latest handshake: 51 seconds ago
  transfer: 7.20 MiB received, 6.82 MiB sent

{% endhighlight %}

- Update Systemd to enable Wireguard at boot:
{% highlight shell %}
systemctl enable wg-quick@wg1
{% endhighlight %}

## Testing
At that point, you should be able to ping from 172.16.255.1 host to
172.16.255.2 and vice versa. If you want to route more subnets on each side,
just add the prefixes with the traditionnal routing tools (route for OpenBSD
and ip for Linux).

## Next steps
The next blog post will be about enabling routing daemons to exchange prefixes between hosts.
