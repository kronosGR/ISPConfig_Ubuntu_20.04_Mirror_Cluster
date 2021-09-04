# Ubuntu 20.04 - ISPConfig - Cluster Mirror (Still working on it!!!)

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

## Master Server Installation
### Upgrade your system
```
sudo -s
apt-get update
apt-get upgrade
reboot

// after reboot run the command as root user. That's why
sudo -s
```

### Change Shell
```
dpk dpkg-reconfigure dash
```
Select **No**

### Disable AppArmor
```
service apparmor stop
update-rc.d -f apparmor remove 
apt-get remove apparmor apparmor-utils
```

### Synchronize System Clock
```
apt-get -y install ntp
```

### Uninstall sendmail
```
service sendmail stop; update-rc.d -f sendmail remove
```
If you received an error "Failed to stop..." then sendmail was not installed

### Install Postfix, Dovecot, MariaDB, rkhunter and binutils
```
apt-get -y install postfix postfix-mysql postfix-doc mariadb-client mariadb-server openssl getmail4 rkhunter binutils dovecot-imapd dovecot-pop3d dovecot-mysql dovecot-sieve sudo patch
```
* Choose -> Internet site 
* Enter -> ns1.lol.me

Edit ports in Postfix:
```
nano /etc/postfix/master.cf
```
Uncomment the lines to be the same with the following:
```
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
```

Restart Postfix
```
service postfix restart
```

Edit MariaDB config file to listen on all interfaces. 
```
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Comment out the bind-address line
```
#bind-address           = 127.0.0.1
```

Set the root password for MariaDB
```
mysql_secure_installation
```
* Press Enter
* Enter y for setting root password
* Enter the new root password
* Re-enter the new root password
* Enter y to all the remaining questions

Set the password authenticatoin method to Native, useful for PHPMyAdmin
```
echo "update mysql.user set plugin = 'mysql_native_password' where user='root';" | mysql -u root
```

Edit debian config file and set the DB's root password twice after **password =**
```
nano /etc/mysql/debian.cnf
```

Edit the scurity limits configuration file
```
nano /etc/security/limits.conf
```
Add the following line to the end of the file
```
mysql soft nofile 65535
mysql hard nofile 65535
```

Configure the Mysql service
```
mkdir /etc/systemd/system/mysql.service.d/
nano /etc/systemd/system/mysql.service.d/limits.conf
```
... and add the following lines
```
[Service]
LimitNOFILE=infinity
```
... restart the services
```
systemctl daemon-reload
service mariadb restart
```

... check that networking is enabled
```
netstat -tap | grep mysql
```








