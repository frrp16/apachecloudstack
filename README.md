# Apache CloudStack 4.18 Installation on Ubuntu 22.04

Kelompok 3 Komputasi Awan 2023/2024

- Abdul Fikih Kurnia (2106731200)
- Eldisja Hadasa (2106640133)
- Farras Rafi Permana (2106700990)
- Muhammad Aqil Muzakky (2106731604)
- Muhammad Irfan Fakhirianto (1806200356)

<br>


## ALL in ONE - Cloudstack Management and Host in One Machine

#### Link Video Rekaman
`
<iframe width="560" height="315" src="https://www.youtube.com/embed/paYmTJH4Bcw?si=axyysODHTmK483X0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
`


### Network Configuration and Additional Tools

|     Home Network      | Gateway   |   Subnet Mask     |
|---------- |----------|------------|
|   192.168.104.0/24       |    192.168.104.1    |    255.255.255.0   |

IP address for
|     Management      | System   |   Public     |
|---------- |----------|------------|
|   192.168.104.23       |    192.168.104.121-130    |    192.168.104.131-140   |


### Set Static IP Address for Management Server
#### Network Configuration with Netplan
#### Rename all existing configuration by adding .bak
```
mv 00-installer-config.yaml 00-installer-config.yaml.bak
```
#### Create new configuration file
```
cat /etc/netplan/01-netcfg.yaml
```
__01-netcfg.yaml__
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.104.23/24]
      routes:
        - to: default
          via: 192.168.104.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp1s0]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

__Apply your network configuration__
```
sudo -i
netplan generate
netplan apply
reboot
```
__Update Your System and Install Some Useful Tools__
```
apt update & upgrade
apt install htop lynx duf -y
apt install bridge-utils
```
__Configure LVM (OPTIONAL)__
```
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

__Install SSH Server and Others Tools__
```
apt-get install openntpd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
#change it to Pa$$w0rd
```

__Enable Root Login (PermitRootLogin)__
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
__Set Timezone__
```
timedatectl set-timezone Asia/Jakarta
```

## APACHE CLOUDSTACK INSTALLATION
### CloudStack Management Server and Everything in one machine
__Reference__
```
https://rohityadav.cloud/blog/cloudstack-kvm/
```

__CloudStack Management Server Setup from SHAPEBLUE ACS 4.18__
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

#update and install cloudstack management and mysql
#it takes very long time

apt-get update -y
apt-get install cloudstack-management mysql-server
```

__CloudStack Usage and Billing (OPTIONAL)__
```
apt-get install cloudstack-usage 
```

__Configure Database --> next time using sed__
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf

#paste below code to mysqld.cnf under [mysqld] block section

[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'

#restart mysql service

systemctl restart mysql
```

__Deploy Database as Root and Then Create "cloud" User with Password "cloud" too__
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.23
```

__Configure Primary and Secondary Storage__
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

__Configure NFS server__
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

### Configure Cloudstack Host with KVM Hypervisor
__Install KVM Host and Cloudstack Agent__
```
apt-get install qemu-kvm cloudstack-agent
```

__Configure Qemu KVM Virtualisation Management (libvirtd)__
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

#configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

__More Configuration to Support Docker and Other Services__
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

__Generate Unique Host ID__
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

__Configure Firewall iptables (OPTIONAL)__
```
NETWORK=192.168.101.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
#just answer yes yes
```

__Disable apparmour on libvirtd__
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

__Launch Management Server and Start Your Cloud__
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
#wait until all services (components) running successfully
```

__Open CloudStack Dashboard__
```
http://192.168.104.23:8080/client
```
Your login username should be __admin__, and your password should be __password__.

![Screenshot 2024-05-29 210247](https://github.com/frrp16/apachecloudstack/assets/87307467/52866c37-fd48-4115-b71f-1c0bee5c96eb)


## Enable XRDP (OPTIONAL) ---> Not working for new UBUNTU
__Reference__
```
#https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```

__Install Desktop XFCE environtment and Remote Desktop XRDP__
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```

__Configure to allow tcp ipv4 listen to 3389. It's a bug only listen to tcp6 --> port=tcp://:3389 --> /etc/xrdp/xrdp.ini__
```
netstat -tulpn | grep xrdp

sed -i.bak 's/^\(port=\).*/\1tcp:\/\/:3389/' /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp
```

## Continue with Instalation (Dashboard)
When inside of the interface, click on __continue with installation__. It will ask for you to change your username and password. Do so and proceed. 
Clicking continue, CloudStack will ask to define a Zone. For this tutorial, we use core zone and choose Advanced core zone type with security group disabled.

#### Configuring Zone
Required fields are the name, IPv4 DNS1, Internal DNS 1, and Hypervisor.

![Screenshot 2024-05-29 211212](https://github.com/frrp16/apachecloudstack/assets/87307467/1b865996-5b58-4f93-8fc2-981f35a81000)

Under network, you can leave your Physical Network as default.

![Screenshot 2024-05-29 211248](https://github.com/frrp16/apachecloudstack/assets/87307467/5505c35b-ef5b-4147-97e6-725abdfff0b4)

Configure the public traffic.

![Screenshot 2024-05-29 211329](https://github.com/frrp16/apachecloudstack/assets/87307467/dc994615-8b15-4fb3-b379-6e8c28dd64bb)

#### Configuring Pod
Configure your pod by specifying pod name and selecting the range of reserved system IP addresses.

![Screenshot 2024-05-29 211427](https://github.com/frrp16/apachecloudstack/assets/87307467/1031ff72-10fe-43f6-86a1-4b205845986c)

Configure the guest network by specifying VLAN range.

![Screenshot 2024-05-29 211455](https://github.com/frrp16/apachecloudstack/assets/87307467/ad6bfea5-107c-4dbe-878a-cc0635d9a71b)

#### Configuring Cluster
Configure cluster of your pod by giving your host the name of your host IP address, followed by the login of root / password of the root user on your host.
![Screenshot 2024-05-29 211514](https://github.com/frrp16/apachecloudstack/assets/87307467/a3a29638-77eb-4033-ab3c-fa6a15400405)
![Screenshot 2024-05-29 211536](https://github.com/frrp16/apachecloudstack/assets/87307467/8d783971-49ab-4f19-bcb8-18a8ccf4a698)

#### Configuring Primary Storage
Configure the primary storage by selecting __nfs__ as the protocol. Then specify host IP address as server and the path is the __/export/primary__ directories.
![Screenshot 2024-05-29 211616](https://github.com/frrp16/apachecloudstack/assets/87307467/c556dabc-6094-46fe-9b03-0e1f793f418d)

#### Configuring Secondary Storage
Configure secondary storage by doing the same way as primary storage, substituting path to __/export/secondary__.
![Screenshot 2024-05-29 211703](https://github.com/frrp16/apachecloudstack/assets/87307467/79b476a0-ff0b-4936-80f8-3f9b9b8a47d2)

Wait a few minute for Cloudstack to install everything. When installation is complete, the dashboard will display statistic of available infrastructure, compute resource, and storage.
![Screenshot 2024-05-30 081851](https://github.com/frrp16/apachecloudstack/assets/87307467/4a432b29-6720-4fa1-b545-8474a4ae72c7)


## Register ISO and Add Instance
This section will show how to upload an ISO and then create a instance utilizing that same ISO within CloudStack. For this example, we will be registering a __Ubuntu 20.04 server__ ISO from https://releases.ubuntu.com/focal/ubuntu-20.04.6-live-server-amd64.iso. 

First, navigate to __Images -> ISOs__. There, click on + Register ISO. There, you will need to input some requested information. The __URL__ is the link to the ISO you want. __Bootable__ lets CloudStack know that this ISO is a bootable disc. __OS Type__ is used to help CloudStack prepare your ISO and makes assumptions to improve performance for the guest. __Extractable__ makes it so that the ISO is available for extraction, Public allows for other users to access the ISO, and __Featured__ will allow for the ISO to be easily selected when creating an instance.

![1Picture1](https://github.com/frrp16/apachecloudstack/assets/87307467/b15b30ec-4cd8-4c74-816f-2268f6c1d2cd)

Once done, CloudStack will begin downloading the ISO. This may take a while. Once done, create an instance utilizing that ISO and install your operating system. 


Instances can be created in __Instances -> Add Instance__. Be sure to select network in the Networks section of instance creation. You can leave the IP address and the MAC address blank if you want CloudStack to automatically configure that for you. Once done, CloudStack will automatically begin creating the instance. 



Click on the name of the Instance and then click on the shell icon in the top right. This will launch a connection to the instance within the browser.


## Install Additional KVM Host
```
sudo -i
apt update
apt upgrade

mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

apt-get update -y
```

__Install KVM Host and Cloudstack Agent__
```
apt-get install qemu-kvm cloudstack-agent
```

__Configure Qemu KVM Virtualisation Management (libvirtd)__
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

#configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
__More Configuration to Support Docker and Other Services__
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

__Generate Unique Host ID__
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

__Firewall disabled (not active) for simplicity__
```
ufw status
#make sure it's inactive
```

__Disable apparmour on libvirtd__
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

__STORAGE SETUP for Additional PRIMARY and SECONDARY__
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

__Configure NFS server__
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

__Enable Root Login (PermitRootLogin)__
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```

## Continue in the cloud management server by adding host
```
#cpu memory increased
#primary and secondary storage increased
```
