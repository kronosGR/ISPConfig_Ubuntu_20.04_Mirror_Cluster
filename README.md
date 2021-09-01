# Ubuntu 20.04 - ISPConfig - Cluster Mirror

## Introduction
Setup with 2 Ubuntu 20.04 servers with ISPConfig web host control panel with master and slave apache server, postfix & dovecot (email), bind dns server and MariaDB database server.

There will be two servers, one master and one slave
____________
**Master Server**

Hostname: ns1.lol.me

IP-Address: 192.168.1.201
______________
**Slave Server**

Hostname: ns2.lol.me

IP-Address: 192.168.1.202
______________


## Install Ubuntu 20.04
Install a server version of Ubuntu 20.04
## Get root privileges
```
    sudo -s
    // enter new password for root user
    sudo passwd root
```

## Install SSH server
```
    sudo apt-get -y install ssh openssh-server
```

## Install nano editor
```
    sudo apt-get install -y nano
```

## Configure the Network
Open the network configuration file
```
    sudo nano /etc/netplan/01-network-manager-all.yaml
```

Change it to look like this below. Change the static IP address you want to use
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp5s0:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.1.201/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Restart the network and apply the changes
```
sudo netplan generate
sudo netplan apply
```

Edit hosts
```
sudo nano /etc/hosts
```

Make it look like this, change it to your wish
```
127.0.0.1       localhost
127.0.1.1       ns1.lol.me ns1

# The following lines are desirable for IPv6 capable hosts
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Change the hostname, in the example ns1 is the host nameservers
```
  sudo echo ns1 > /etc/hostname
  sudo hostname ns1

  // verify the changes
  hostname
  hostname -f
```


