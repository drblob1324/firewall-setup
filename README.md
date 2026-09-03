# firewall-setup

The goal of this project is to set up a Linux system running OpenSUSE Tumbleweed to act as a firewall between an insecure legacy system (OS X 10.4) and the wider network, as well as a dashboard hosted over the LAN for configuration - as a DIY IPFire or pfSense alternative, albeit less featured.

### 1. Setting up the connection
For the purposes of this, the outside world is connected through 'wlo1' on 192.168.1.xxx and the network adapter for the legacy system is enp4s0f3u2. 

Running ```sudo nmcli connection show``` presented 'wlo1' and 'Wired connection 1' as well as a loopback address. 
To configure the Ethernet connection, on the OS X legacy machine I used the built in 'Network' panel in System Preferences to disable AirPort Wireless and enable the Ethernet connection with a manual IP of ```192.168.2.2``` and a router of ```192.168.2.1``` which is the IP address of the Linux machine I wish to use a firewall. On the Linux machine I set a static IP using ```sudo nmcli connection modify 'Wired connection 1' ipv4.addresses 192.168.2.1 ipv4.method manual ipv4.gateway 192.168.1.254 ipv4.dns '192.168.1.254'``` where 192.168.1.254 is the IP of the router in the outside network. I was, of course, immediately punished for forgetting the /24 after the ```ipv4.addresses``` so I had to delete the connection and try again.

To set up the connection I read https://doc.opensuse.org/documentation/leap/archive/15.3/security/html/book-security/cha-security-firewall.html and decided what needed doing fell into four parts. \
__1. IP forwarding \
2. NAT/MASQUERADE \
3. FORWARD rules \
4. and the default-deny policy__

I enabled routing with ```sysctl -w net.ipv4.ip_forward=1``` and then moved on to firewall setup

### 2. Setting up and configuring the firewall
I wanted my firewall to be configurable from the network portal, so ideally everything would live in a single file, or directory. As a base, I am going to use ```nftables```, since that's as low-level as I feel like I can properly configure without a huge learning curve
