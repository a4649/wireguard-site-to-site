# WireGuard Site-to-Site VPN

![diagram](images/diagram.png)

1. Pre-requisites
2. Virtual Machines
3. OS and networking
4. Wireguard configuration
5. Site routing configuration

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

Install wireguard-tools package

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

## Wireguard configuration

Go to wireguard directory and generate public and private key for each virtual machine

cd /etc/wireguard/
umask 077
wg genkey | tee privatekey | wg pubkey > publickey

Copy the configuration files 

