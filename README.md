#Tunlr Clone#
To build a tunlr or UnoTelly or unblock-us.com (or other DNS-based services) clone on the cheap, you need to invest in a VPS.
For the purposes of this discussion, I am assuming that you will be using this for watching US geo-locked content.
##VPS Provider Specific Terminology##
My VPS provider is [buyvm](http://buyvm.net/). I have an OpenVZ 128m plan with them hosted in New York. So, the venet0 references that
you will see pertain to that VPS provider.
##Disclaimer##
This information is provided as is without warranty of any kind, either express or implied, including but not limited to the implied warranties of merchantability and fitness for a particular purpose. In no event shall the author be liable for any damages whatsoever including direct, indirect, incidental consequential, loss of business profits, or special damages.

If you leave your DNS server or Squid proxy server wide open to abuse, that's on your own head. Take precautions in this regard. Proceed at your own risk.

Also this is not meant to be a hold-your-hand start from scratch tutorial. Therefore, some level of Linux expertise is necessary.
##Background##
Basically we are interested in proxying content only for certain domains. The actual streaming media sits on CDN
networks and is usually not geo-locked. The amount of proxying we'll end up doing will be relatively
insignificant compared to a VPN-based setup.
[![How Tunlr Cloning works](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)
##US IP Address##
Your VPS provider must provide you with a US IP address
##Tomato based router##
Since you will be changing DNS servers to point to your "own" DNS, it makes sense to run `dnsmasq` on your router,
therefore having a Tomato capable router is preferable.

Following is my `dnsmasq` configuration on my Tomato-based router (running a Toastman build):
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
server=/hulu.com/199.x.x.x
server=/huluim.com/199.x.x.x
# tunlr for US networks
# cbs works with link.theplatform.com
server=/abc.com/abc.go.com/199.x.x.x
server=/fox.com/link.theplatform.com/199.x.x.x
server=/nbc.com/nbcuni.com/199.x.x.x
server=/ip2location.com/199.x.x.x
# espn3 
server=/broadband.espn.go.com/199.x.x.x

# Google
server=8.8.8.8
server=8.8.4.4
# OpenDNS
#server=208.67.222.222
#server=208.67.220.220
```
In essence, I am forwarding DNS queries to my VPS only for the specified domains. Everything else goes to Google DNS
(or can easily go to your ISP DNS).

##Your own DNS Server##
I am running bind9 on my VPS to override the DNS resolution for the entire domains mentioned in the Tomato-based router configuration above.
The plan is to send the external IP address of my VPS as the resolved IP address for any of those domains.

Once the web traffic hits my VPS, I use iptables to redirect port 80 traffic to squid running on port 8xxx.

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
    172.x.x.x;        // local venet0:1 internal IP here
    127.0.0.1;
    173.x.x.x;        // Your ISP IP here (cable/DSL)
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
    IN  NS  ns1
; specify nameserver IP address
ns1 IN  A   199.y.y.y                ; external IP from venet0:0
; provide IP address for domain itself
@   IN  A   199.y.y.y                ; external IP from venet0:0
; resolve everything with the same IP address as ns1
*   IN  A   199.y.y.y                 ; external IP from venet0:0
```
When you discover a new domain that you want to "master", simply add it to the `zones.override` file and restart bind9.

##Squid##
`/etc/squid3/squid.conf`
```squid
# grep '^[^#]' /etc/squid3/squid.conf
acl manager proto cache_object
acl localhost src 127.0.0.1/32 ::1
acl to_localhost dst 127.0.0.0/8 0.0.0.0/32 ::1
acl trusted src 172.x.x.x 173.y.y.y        # internal IP from venet0:1 and ISP IP (Cable/DSL)
acl SSL_ports port 443
acl Safe_ports port 80  	# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http
acl CONNECT method CONNECT
http_access allow manager localhost
http_access deny manager
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow trusted
http_access allow localhost
http_access deny all
http_port 0.0.0.0:8128 transparent
hierarchy_stoplist cgi-bin ?
debug_options ALL,3
coredump_dir /var/spool/squid3
cache deny all
refresh_pattern ^ftp:    	1440	20%	10080
refresh_pattern ^gopher:	1440	0%	1440
refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
refresh_pattern .		0	20%	4320
request_header_access Proxy-Connection deny all
request_header_access X-Forwarded-For deny all
request_header_access Connection deny all
request_header_access Via deny all
forwarded_for off
```
##Iptables##
`172.x.x.x` is the venet0:1 internal IP address. 

For the `filter` table (which is the default):
```bash
iptables -A INPUT -i venet0 -d 172.x.x.x -p tcp -m tcp --dport 8128 -j ACCEPT
```
For the `nat` table:
```bash
iptables -t nat -A PREROUTING -i venet0 -p tcp --dport 80 -j DNAT --to 172.x.x.x:8128
```