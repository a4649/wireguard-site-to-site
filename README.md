# WireGuard Site-to-Site VPN

![diagram](images/diagram.png)

1. Pre-requisites
2. Virtual Machines
3. OS and networking
4. Wireguard configuration
5. Connect and check
6. Site routing configuration

## Pre-requisites

* Public IP Address or dynamic DNS on WAN links
* Hypervisor in each site in order to create the linux virtual machine (*not mandatory)
* Management access to the routers configuration in each site to configure port forwarding and static routes

*You can use a physical computer/server running linux instead of a virtual machine

## Virtual Machines

Again, is not mandatory to use virtual machines, you can use an RaspberryPi or an old computer (with new SSD HD!)

Setup two virtual machines with:
* 2 vCPU
* 4GB of RAM
* 15GB of disk space
* Bridge networking

These values may need to be adapted depending on your needs. This setup works fine for me with the following use case:
* ~20 machines in site B acessing storage server in site A
* 250Mbps upload internet connection in site A
* ~900GBytes/day

Download the minimal version (do not install GUI) ISO of any Linux distribution and install the basic OS. In this guide I will use an RHEL8 based distro - CentOS8. The commands and paths should be the same for Oracle Linux 8, Scientific Linux 8,Rocky Linux or other RHEL8 based distro). Ensure to give static IP Address for the local ethernet interface of the virtual machine.  

## OS and networking

Alfet install the base OS, install the EPEL repository, update the system and reboot

### Upgrading kernel using ElRepo mainline stable kernel

RHEL8 comes with 4.18.X kernel version, in order to use wireguard we need to upgrade it to 5.6 or higher, other way could be installing the kernel modules in the official kernel ( not followed in this guide)

If you are using other linux distribution with kernel 5.6 or higher you can skip this steps

Import repository public key:

rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

Add repository:

dnf install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm

Install kernel:

dnf --enablerepo=elrepo-kernel install kernel-ml

Set it by default when booting:

grub2-set-default 0

Reboot the VM

### Once it boots check if the wireguard module is present

modinfo wireguard

Install wireguard-tools package:

dnf install wireguard-tools

### Enable IPv4 forwarding and disable IPv6

Using any text editor edit the file /etc/sysctl.conf and add the following lines:

net.ipv4.ip_forward = 1

net.ipv6.conf.all.disable_ipv6 = 1

net.ipv6.conf.default.disable_ipv6 = 1

net.ipv6.conf.lo.disable_ipv6 = 1

Apply and restart network:

sysctl -p

systemctl restart NetworkManager

Add firewalld rules:

firewall-cmd --add-port=15380/tcp

firewall-cmd --add-port=15380/udp

firewall-cmd --zone=public --add-masquerade

firewall-cmd --runtime-to-permanent

Ensure you have the port 15380 TCP and UDP forwarding to virtual machines IP Address in your router/gateway in both sites

## Wireguard configuration

Generate public and private key for each virtual machine:

wg genkey | tee privatekey | wg pubkey > publickey

Move the keys to the wireguard folder:

mv publickey /etc/wireguard/

mv privatekey /etc/wireguard/

Clone configuration files: 

git clone https://github.com/a4649/wireguard-site-to-site.git

### For site-A:

Edit the wireguard-site-to-site/site-A/wg0.conf file:

At line number 4 replace 'site-A-private-key' with exact content of /etc/wireguard/privatekey

At line number 9 replace 'site-B-public-key' with exact content of /etc/wireguard/publickey in site-B virtual machine

Copy it to wireguard folder:

cp -v wireguard-site-to-site/site-A/wg0.conf /etc/wireguard/

### For site-B:

Edit the wireguard-site-to-site/site-B/wg0.conf file:

At line number 3 replace 'site-B-private-key' with exact content of /etc/wireguard/privatekey

At line number 9 replace 'site-A-public-key' with exact content of /etc/wireguard/publickey in site-A virtual machine

At line number 11 replace 'site-A-WAN_IP_ADDRESS' with site A WAN IP Address

Copy it to wireguard folder:

cp -v wireguard-site-to-site/site-B/wg0.conf /etc/wireguard/

## Connect and check

To start the connection(in both sites):

wg-quick up wg0

Stop connection: 

wg-quick down wg0

Checking connection state:

wg show

Start connection when virtual machine boot:

systemctl enable wg-quick@wg0.service

### Check from site-A:

tracepath 192.168.20.200

### Check from site-B:

tracepath 192.168.10.200

## Site routing configuration

Add an static route in your routers/gateways or servers with remote subnets as destination with Virtual Machine IP addresse as next hop

### Site A examples:

Linux server (temporary, you need to add it in a network-script to be permanent): 

ip route add 192.168.20.0/24 via 192.168.10.200 dev eth0

Windows server: 

route -P ADD 192.168.20.0 MASK 255.255.255.0 192.168.10.200

Cisco router: 

ip route 192.168.20.0 255.255.255.0 192.168.10.200

Unifi Gateway: 

Settings > Routing & Firewall > Static Routes 
