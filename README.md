#Tunlr Clone#
To build a tunlr or UnoTelly or unblock-us.com (or other DNS-based
services) clone on the cheap, you need to invest in a VPS.  For the
purposes of this discussion, I am assuming that you will be using
this for watching US geo-locked content.

##US IP Address##
Your VPS provider must provide you with an exteral IP address with
presence in the US.

##VPS Provider Specific Terminology##
My VPS provider is [buyvm](http://buyvm.net/).  I have an OpenVZ
128m plan with them hosted in New Jersey (running Debian 7).  So, the
venet0 references that you will see pertain to that VPS provider.

##Disclaimer##
This information is provided as is without warranty of any kind,
either express or implied, including but not limited to the implied
warranties of merchantability and fitness for a particular purpose.
In no event shall the author be liable for any damages whatsoever
including direct, indirect, incidental consequential, loss of
business profits, or special damages.

If you leave your DNS server or HTTPS-SNI-Proxy server wide open to
abuse, that's on your own head.  Take precautions in this regard.
Proceed at your own risk.

Also this is not meant to be a hold-your-hand start from scratch
tutorial.  Therefore, some level of Linux expertise is necessary.

##Background##
Basically we are interested in proxying content only for certain
geo-locked domains.  The actual streaming media sits on CDNs and is
usually not geo-locked.  The amount of proxying we'll end up doing
will be relatively insignificant compared to a VPN-based setup (where
everything must be proxied). The outlined solution will typically
result in significant bandwidth savings.

[![How Tunlr Cloning works](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)

User browses to Hulu homepage. Behind the scenes, this triggers the
following sequence of events:

1. Browsing device asks for the IP address of www.hulu.com (using DNS).
1. Since the router is running `dnsmasq`, it selectively sends the DNS
   query for www.hulu.com to DNS server running on the VPS.
1. The VPS DNS server responds with the IP address of VPS SNI Server as the
   authorative answer for the DNS query.
1. Router sends resolved IP address back to browsing device.
1. Browsing device sends a request for content for www.hulu.com.
1. VPS SNI Server sends a request for content to www.hulu.com.
1. Since the VPS SNI Server has an IP presence in USA, www.hulu.com
   responds with proper content.
1. VPS SNI Server proxies the content back to the browsing device

It is pertinent to point out that if your router is dnsmasq-capable,
then, you could bypass steps 2 and 3 (go directly from step 1 straight
to 4) by having `dnsmasq` resolve the address of relevant domains.

##Routers capable of dnsmasq (e.g., those running Tomato)##
If the stock firmware of your router supports `dnsmasq` or if your
router is capable of running Tomato, then, you can use `dnsmasq` for
your DNS-resolution needs.

Following is my `dnsmasq` configuration on my Tomato-based router
(running a Toastman build):
`Advanced -> DHCP/DNS -> Dnsmasq Custom configuration`
```bash
# Never forward plain names (without a dot or domain part)
domain-needed
# Never forward addresses in the non-routed address spaces.
bogus-priv

# If you don't want dnsmasq to read /etc/resolv.conf or any other
# file, getting its servers from this file instead (see below), then
# uncomment this.
no-resolv

# If you don't want dnsmasq to poll /etc/resolv.conf or other resolv
# files for changes and re-read them then uncomment this.
no-poll

# tunlr for hulu
address=/hulu.com/199.x.x.x
address=/huluim.com/199.x.x.x
address=/netflix.com/199.x.x.x
# tunlr for US networks
# cbs works with link.theplatform.com
address=/abc.com/abc.go.com/199.x.x.x
address=/fox.com/link.theplatform.com/199.x.x.x
address=/nbc.com/nbcuni.com/199.x.x.x
address=/pandora.com/199.x.x.x
address=/ip2location.com/199.x.x.x
# espn3 
address=/broadband.espn.go.com/199.x.x.x

# Google
server=8.8.8.8
server=8.8.4.4
# OpenDNS
#server=208.67.222.222
#server=208.67.220.220
```
`199.x.x.x` is the IP address of my VPS server (where my SNI proxy server will
also be running). See next section(s).

In essence, I am resolving DNS queries to point to my VPS only for the
specified domains.  Everything else goes to Google DNS (or can
easily go to your ISP DNS). This means that CDN-hosted media DNS
resolution will not be going to our Linux VPS.

##Routers incapable of running dnsmasq##
If you have a router which does not support running `dnsmasq` (either
via the stock firmware or is not Tomato-capable), then, you will have to
point all of your DNS queries to a DNS server running in your
control on your VPS.

##Your own DNS Server running on VPS##
In the situation where my router is incapable of running `dnsmasq` I
will have to run a DNS server (e.g., bind9) on my VPS. The plan is to send the external IP address of my VPS as the resolved IP address for any of the relevant domains.
Everything else will be forwarded to another DNS for resolution.

Once the web traffic hits my VPS, I use iptables to limit access to
traffic provided by
[HTTPS-SNI-Proxy](https://github.com/dlundquist/HTTPS-SNI-Proxy)
running on port 80/443 (since currently HTTPS-SNI-Proxy does not have ACL
capability).

Here is the bind9 config:

`/etc/bind/named.conf.options`
```nginx
options {
    directory "/var/cache/bind";
	forwarders {
        # these are the DNS servers from the VPS provider (look in /etc/resolv.conf if yours are different)
		199.195.255.68;
		199.195.255.69;
	};

	auth-nxdomain no;    # conform to RFC1035
	listen-on-v6 { any; };
	allow-query { trusted; };
	allow-recursion { trusted; };
	recursion yes;
	dnssec-enable no;
	dnssec-validation no;
};
```
`/etc/bind/named.conf.local`
```nginx
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

include "/etc/bind/rndc.key";

acl "trusted" {
    172.y.y.y;        // local venet0:17 internal IP here
    127.0.0.1;
    173.z.z.z;        // Your ISP IP here (cable/DSL)
};

include "/etc/bind/zones.override";

logging {
    channel bind_log {
        file "/var/log/named/named.log" versions 5 size 30m;
        severity info;
        print-time yes;
        print-severity yes;
        print-category yes;
    };
    category default { bind_log; };
    category queries { bind_log; };
};
```
`/etc/bind/zones.override`
```nginx
zone "hulu.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "huluim.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "netflix.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "abc.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "abc.go.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "fox.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "link.theplatform.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "nbc.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "nbcuni.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "pandora.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "broadband.espn.go.com." {
    type master;
    file "/etc/bind/db.override";
};
zone "ip2location.com." {
    type master;
    file "/etc/bind/db.override";
};
```
`/etc/bind/db.override`
```
;
; BIND data file for overridden IPs
;
$TTL  86400
@   IN  SOA ns1 root (
            2012100401  ; serial
            604800      ; refresh 1w
            86400       ; retry 1d
            2419200     ; expiry 4w
            86400       ; minimum TTL 1d
            )

; need atleast a nameserver
@   IN  NS  ns1
; specify nameserver IP address
ns1 IN  A   199.x.x.x                ; external IP from venet0:0
; provide IP address for domain itself
@   IN  A   199.x.x.x                ; external IP from venet0:0
; resolve everything with the same IP address as ns1
*   IN  A   199.x.x.x                 ; external IP from venet0:0
```
When you discover a new domain that you want to "master", simply
add it to the `zones.override` file and restart bind9.

##HTTPS-SNI-Proxy##
Install according to the instructions on
[HTTPS-SNI-Proxy](https://github.com/dlundquist/HTTPS-SNI-Proxy)

`/etc/sniproxy.conf`
```sniproxy.conf
# grep '^[^#]' /etc/sniproxy.conf
user daemon
pidfile /var/tmp/sniproxy.pid
listener 172.y.y.y 80 {
    proto http
}
listener 172.y.y.y 443 {
    proto tls
}
table {
    (hulu|huluim)\.com *
    abc\.(go\.)?com *
    (nbc|nbcuni)\.com *
    netflix\.com *
    ip2location\.com *
}
```
##Iptables##
`172.y.y.y` is the venet0:17 internal IP address. `173.x.x.x` is your ISP
address provided by Cable or DSL.

For the `filter` table (which is the default):
```bash
iptables -A INPUT -i venet0 -s 173.x.x.x -d 172.y.y.y -p tcp -m tcp --dport 80 -j ACCEPT
iptables -A INPUT -i venet0 -s 173.x.x.x -d 172.y.y.y -p tcp -m tcp --dport 443 -j ACCEPT
```
For the `nat` table:
```bash
iptables -t nat -A PREROUTING -i venet0 -p tcp --dport 80 -j DNAT --to 172.y.y.y
iptables -t nat -A PREROUTING -i venet0 -p tcp --dport 443 -j DNAT --to 172.y.y.y
```

##Limitations##
At the time of writing, this procedure does not work in at least the following situations:

1. Any devices which do not support the use of SNI (Server Name Indication) during SSL 3.0 handshake, e.g.:
    1. Netflix on Chromecast, PlayStation, XBox, Nexus 7 FHD, some LG smart TVs, etc.
