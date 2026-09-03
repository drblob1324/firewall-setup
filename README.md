# firewall-setup

The goal of this project is to set up a Linux system running OpenSUSE Tumbleweed to act as a firewall between an insecure legacy system (OS X 10.4) and the wider network, as well as a dashboard hosted over the LAN for configuration - as a DIY IPFire or pfSense alternative, albeit less featured.

## 1. Setting up the connection
For the purposes of this, the outside world is connected through 'wlo1' on 192.168.1.xxx and the network adapter for the legacy system is enp4s0f3u2. 

Running ```sudo nmcli connection show``` presented 'wlo1' and 'Wired connection 1' as well as a loopback address. 
To configure the Ethernet connection, on the OS X legacy machine I used the built in 'Network' panel in System Preferences to disable AirPort Wireless and enable the Ethernet connection with a manual IP of ```192.168.2.2``` and a router of ```192.168.2.1``` which is the IP address of the Linux machine I wish to use a firewall. On the Linux machine I set a static IP using ```sudo nmcli connection modify 'Wired connection 1' ipv4.addresses 192.168.2.1 ipv4.method manual ipv4.gateway 192.168.1.254 ipv4.dns '192.168.1.254'``` where 192.168.1.254 is the IP of the router in the outside network. I was, of course, immediately punished for forgetting the /24 after the ```ipv4.addresses``` so I had to delete the connection and try again.

To set up the connection I read https://doc.opensuse.org/documentation/leap/archive/15.3/security/html/book-security/cha-security-firewall.html and decided what needed doing fell into four parts. 

__1. IP forwarding \
2. NAT/MASQUERADE \
3. FORWARD rules \
4. and the default-deny policy__

I enabled routing with ```sysctl -w net.ipv4.ip_forward=1``` and then moved on to firewall setup

## 2. Setting up and configuring the firewall
I wanted my firewall to be configurable from the network portal, so ideally everything would live in a single file, or directory. As a base, I am going to use ```nftables```, since that's as low-level as I feel like I can properly configure without a huge learning curve

Currently, ```sudo nftables list ruleset``` gives no output, since nftables isn't configured yet. To configure nftables, I found https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/getting-started-with-nftables_configuring-and-managing-networking to be very helpful. \
I started by adding an nft table with ```sudo nft add table inet firewall``` following the
> nft add table <table_address_family> <table_name>

syntax. Then I needed to add a chain, which are containers for rules. 
> nft add chain <table_address_family> <table_name> <chain_name> { type <type> hook <hook> priority <priority> \; policy <policy> \; }

To set up forwarding I used ```sudo nft add chain inet firewall forward { type filter hook forward priority filter; policy drop; }```. This got me to 
```
user@system ~ $ sudo nft list ruleset
table inet firewall {
	chain forward {
		type filter hook forward priority filter; policy drop;
	}
}
```
The syntax 
> nft add rule <table_address_family> <table_name> <chain_name> <rule>

allowed me to create the rules ```sudo nft add rule inet firewall forward iifname "enp4s0f3u2" oifname "wlo1" ip saddr 192.168.2.0/24 accept``` and ```sudo nft add rule inet firewall forward iifname "wlo1" oifname "enp4s0f3u2" ip daddr 192.168.2.0/24 ct state established,related accept```

This means that connections from the LAN to WiFi are allowed (```iifname``` of the Ethernet adaptor and ```oifname``` of the WiFi are configured to ```accept```) and connections from WiFi to LAN are blocked (```policy drop;```) unless they are ```state established``` and ```related``` to ```accept```) so traffic can come in but only if the legacy system first established the connection. \
This means that outdated versions of services like SSH and SMB cannot be exploited by attackers and makes the device less easy to detect on port scans on the wider network, but still allows network traffic in for things like a web request where things have to be returned from the HTTP GET request.

### Configuring NAT
NAT is important because packets can make it *out* of the legacy system, but not back in. We can see this with tcpdump. Doing ```ping -c 3 192.168.1.254``` on the firewalled legacy system and watching tcpdump on the Ethernet device leads to 
```
user@system ~ $ sudo tcpdump -ni enp4s0f3u2 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:23:58.558297 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 0, length 64
11:23:59.558837 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 1, length 64
11:24:00.558520 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 2, length 64
```
and watching it on the WiFi devices shows 
```
user@system ~ $ sudo tcpdump -ni wlo1 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:23:58.558297 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 0, length 64
11:23:59.558837 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 1, length 64
11:24:00.558520 IP 192.168.2.2 > 192.168.1.254: ICMP echo request, id 56064, seq 2, length 64
```
meaning that the network is successfully passing the Ethernet traffic to the WiFi interface. So forwarding is working.

**NAT** or **Network Address Translation** allows the packets to be returned through the firewall so the ```ping``` command can succeed, as can HTTP requests, etc... \
I found https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/security_guide/sec-configuring_nat_using_nftables to be helpful when configuring NAT.

First, I added a NAT table to nftables, with ```sudo nft add table ip nat``` and then added a chain; ```sudo nft 'add chain ip nat postrouting { type nat hook postrouting priority srcnat; policy accept; }'```\
Then the masquerading rule ```sudo nft add rule ip nat postrouting oifname "wlo1" ip saddr 192.168.2.0/24 masquerade```

So my final base ruleset ended up as
```
table inet firewall {
	chain forward {
		type filter hook forward priority filter; policy drop;
		iifname "enp4s0f3u2" oifname "wlo1" ip saddr 192.168.2.0/24 accept
		iifname "wlo1" oifname "enp4s0f3u2" ip daddr 192.168.2.0/24 ct state established,related accept
	}
}
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "wlo1" ip saddr 192.168.2.0/24 masquerade
	}
}
```

### Testing it works

We can test if *NAT* is working with a tcpdump again. On the legacy, firewalled system I did ```ping -c 3 8.8.8.8``` and ```sudo tcpdump -ni enp4s0f3u2 icmp``` on the Linux system. \
The ping worked and finished with ```0% packet loss``` and the ```tcpdump``` shows both the outgoing ```11:36:19.348573 IP 192.168.2.2 > 8.8.8.8: ICMP echo request, id 57344, seq 0, length 64```
and incoming ```11:36:19.357112 IP 8.8.8.8 > 192.168.2.2: ICMP echo reply, id 57344, seq 0, length 64``` packets.

We can test if the *firewall* is working with a simple ping from outside. 
```
user@outsideSystem ~ % ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
Request timeout for icmp_seq 3
^C
--- 192.168.2.2 ping statistics ---
5 packets transmitted, 0 packets received, 100.0% packet loss
```
meaning the firewall is indeed working. (I did have a bit of a worry when I realised that the ping was working, and then realised I was still SSH'd into the firewall system and so it wasn't "outside traffic")s
