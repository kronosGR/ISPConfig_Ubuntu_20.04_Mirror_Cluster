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


### Install Amasivd-new, SpamAssassin and Clamav
```
apt-get -y install amavisd-new spamassassin clamav clamav-daemon unzip bzip2 arj nomarch lzop cabextract apt-listchanges libnet-ldap-perl libauthen-sasl-perl clamav-docs daemon libio-string-perl libio-socket-ssl-perl libnet-ident-perl zip libnet-dns-perl postgrey
```

Stop SpamAssassin
```
service spamassassin stop
update-rc.d -f spamassassin remove
```

Start Clamav
```
freshclam
service clamav-daemon start
```
*Error for first freshclam at the first use can be ignored*

### Install Apache, PHP, phpMyAdmin, FCGI, SuExec, Pear
```
apt-get -y install apache2 apache2-doc apache2-utils libapache2-mod-php php7.4 php7.4-common php7.4-gd php7.4-mysql php7.4-imap phpmyadmin php7.4-cli php7.4-cgi libapache2-mod-fcgid apache2-suexec-pristine php-pear libruby libapache2-mod-python php7.4-curl php7.4-intl php7.4-pspell php7.4-sqlite3 php7.4-tidy php7.4-xmlrpc php7.4-xsl memcached php-memcache php-imagick php7.4-zip php7.4-mbstring php-soap php7.4-soap php7.4-opcache php-apcu php7.4-fpm libapache2-reload-perl
```
* Choose **apache2**
* Choose **Yes** to configure phpmyadmin with dbconfig-common
* **Press Enter** for password for phpmyadmin
  
Enable some Apache modules
```
a2enmod suexec rewrite ssl actions include cgi alias proxy_fcgi
a2enmod dav_fs dav auth_digest headers
```

... disable HTTPOXY
```
nano /etc/apache2/conf-available/httpoxy.conf
```

... add the following 
```
<IfModule mod_headers.c>
    RequestHeader unset Proxy early
</IfModule>
```

... enable the config file
```
a2enconf httpoxy
```

... restart Apache
```
service apache2 restart
```

### Install Let's Encrypt
```
apt-get install -y certbot
```

### Install Mailman
```
apt-get -y install mailman
```
* Select at least one language
* **OK** on missing site list

... create a mail list
```
newlist mailman
```
* Enter admin email address
* Enter a mailman password
* Press **ENTER**

... open tha aliases file  
```
nano /etc/aliases
```

... and add the lines
```
## mailman mailing list
mailman:              "|/var/lib/mailman/mail/mailman post mailman"
mailman-admin:        "|/var/lib/mailman/mail/mailman admin mailman"
mailman-bounces:      "|/var/lib/mailman/mail/mailman bounces mailman"
mailman-confirm:      "|/var/lib/mailman/mail/mailman confirm mailman"
mailman-join:         "|/var/lib/mailman/mail/mailman join mailman"
mailman-leave:        "|/var/lib/mailman/mail/mailman leave mailman"
mailman-owner:        "|/var/lib/mailman/mail/mailman owner mailman"
mailman-request:      "|/var/lib/mailman/mail/mailman request mailman"
mailman-subscribe:    "|/var/lib/mailman/mail/mailman subscribe mailman"
mailman-unsubscribe:  "|/var/lib/mailman/mail/mailman unsubscribe mailman"
```

... then 
```
newaliases
service postfix restart
```

... enable the Mailman Apache configuration
```
ln -s /etc/mailman/apache.conf /etc/apache2/conf-available/mailman.conf
```

... activate the configuration and restart apache and mailman
```
a2enconf mailman
service apache2 restart
service mailman start
```

### Install PureFTPd and Quota
```
apt-get -y install pure-ftpd-common pure-ftpd-mysql quota quotatool
```

... open the file
```
nano /etc/default/pure-ftpd-common
```
... check the following line to be the same
```
STANDALONE_OR_INETD=standalone
[...]
VIRTUALCHROOT=true
```

... allow FTP and TLS sessions
```
echo 1 > /etc/pure-ftpd/conf/TLS
```
... and create a SSL sertificate
```
mkdir -p /etc/ssl/private/
openssl req -x509 -nodes -days 7300 -newkey rsa:2048 -keyout /etc/ssl/private/pure-ftpd.pem -out /etc/ssl/private/pure-ftpd.pem
```
... change the permissions of SSL certificate and restart FTPd
```
chmod 600 /etc/ssl/private/pure-ftpd.pem
service pure-ftpd-mysql restart
```

Edit fstab file to add quota
```
nano /etc/fstab
```

... add **,usrjquota=quota.user,grpjquota=quota.group,jqfmt=vfsv0** after remount-ro at the root partiion
```
 / ext4 errors=remount-ro,usrjquota=quota.user,grpjquota=quota.group,jqfmt=vfsv0 0 1
```
...enable quota
```
mount -o remount /
quotacheck -avugm
quotaon -avug
```

### Install BIND DNS Server
```
apt-get -y install bind9 dnsutils haveged
```

... enable and start haveged Daemon
```
systemctl enable haveged
systemctl start haveged
```

### Install Vlogger, Webalizer, AWStats and GoAccess
```
echo "deb https://deb.goaccess.io/ $(lsb_release -cs) main" | sudo tee -a /etc/apt/sources.list.d/goaccess.list

wget -O - https://deb.goaccess.io/gnugpg.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/goaccess.gpg add -

apt-get -y install vlogger webalizer awstats geoip-database libclass-dbi-mysql-perl goaccess
```

... edit awstats
```
nano /etc/cron.d/awstats
```
... and comment out everything
```
#MAILTO=root

#*/10 * * * * www-data [ -x /usr/share/awstats/tools/update.sh ] && /usr/share/awstats/tools/update.sh

# Generate static reports:
#10 03 * * * www-data [ -x /usr/share/awstats/tools/buildstatic.sh ] && /usr/share/awstats/tools/buildstatic.sh
```

### Install Jailkit
```
apt-get -y install jailkit
```

### Install fail2ban and UFW
```
apt-get -y install fail2ban
```
... create the file /etc/fail2ban/jail.local
```
nano /etc/fail2ban/jail.local
```
... and add the following
```
[pure-ftpd]
enabled  = true
port     = ftp
filter   = pure-ftpd
logpath  = /var/log/syslog
maxretry = 3

[dovecot]
enabled = true
filter = dovecot
action = iptables-multiport[name=dovecot-pop3imap, port="pop3,pop3s,imap,imaps", protocol=tcp]
logpath = /var/log/mail.log
maxretry = 5

[postfix]
enabled  = true
port     = smtp
filter   = postfix
logpath  = /var/log/mail.log
maxretry = 3
```

... restart fail2ban and install UFW
```
service fail2ban restart
apt-get install ufw
```