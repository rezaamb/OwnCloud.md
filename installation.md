the ubuntu mechine must be in 22.04

##Preparation

###Set Your Domain Name

```bash
my_domain="Your.Domain.tld"
echo $my_domain

hostnamectl set-hostname $my_domain
hostname -f
```
###Update Your System

First, ensure that all the installed packages are entirely up to date and that PHP is available in the APT repository. To do so, follow the instructions below:
```bash
apt update && \
  apt upgrade -y
```
### Create the occ Helper Script 

Create a helper script to simplify running occ commands:

```bash
FILE="/usr/local/bin/occ"
cat <<EOM >$FILE
#! /bin/bash
cd /var/www/owncloud
sudo -E -u www-data /usr/bin/php /var/www/owncloud/occ "\$@"
EOM
```
Make the helper script executable:

```bash
chmod +x $FILE
```

##Install the Required Packages

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update && sudo apt upgrade

apt install -y \
  apache2 \
  libapache2-mod-php7.4 \
  mariadb-server openssl redis-server wget \
  php7.4 php7.4-imagick php7.4-common php7.4-curl \
  php7.4-gd php7.4-imap php7.4-intl php7.4-json \
  php7.4-mbstring php7.4-gmp php7.4-bcmath php7.4-mysql \
  php7.4-ssh2 php7.4-xml php7.4-zip php7.4-apcu \
  php7.4-redis php7.4-ldap php-phpseclib
```

##Install smbclient php Module

If you want to connect to external storage via SMB you need to install the smbclient php module.

```bash
apt-get install -y php7.4-smbclient
echo "extension=smbclient.so" > /etc/php/7.4/mods-available/smbclient.ini
phpenmod smbclient
systemctl restart apache2
```

Check if it was successfully activated:
```bash
php -m | grep smbclient
```

This should show the following output:

```bash
libsmbclient
smbclient
```

##Install the Recommended Packages

Additional useful tools helpful for debugging:

```bash
apt install -y \
  unzip bzip2 rsync curl jq \
  inetutils-ping  ldap-utils\
  smbclient
```

#Configure Apache

Create a Virtual Host Configuration:

```bash
FILE="/etc/apache2/sites-available/owncloud.conf"
cat <<EOM >$FILE
<VirtualHost *:80>
# uncommment the line below if variable was set
#ServerName \$my_domain
DirectoryIndex index.php index.html
DocumentRoot /var/www/owncloud
<Directory /var/www/owncloud>
  Options +FollowSymlinks -Indexes
  AllowOverride All
  Require all granted

 <IfModule mod_dav.c>
  Dav off
 </IfModule>

 SetEnv HOME /var/www/owncloud
 SetEnv HTTP_HOME /var/www/owncloud
</Directory>
</VirtualHost>
EOM
```

Test the Configuration
```bash
apachectl -t
```

At this point, the following output is expected:

```bash
apachectl -t
AH00112: Warning: DocumentRoot [/var/www/owncloud] does not exist
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

The first warning will be resolved after ownCloud is installed. The second message can be resolved with following command. Check that the entry is only present once in the apache2.conf file:

```bash
echo "ServerName $my_domain" >> /etc/apache2/apache2.conf
```

Enable the Virtual Host Configuration

```bash
a2dissite 000-default
a2ensite owncloud.conf
```

##Configure the Database

Ensure transaction-isolation level is set and performance_schema on.

```bash
sed -i "/\[mysqld\]/atransaction-isolation = READ-COMMITTED\nperformance_schema = on" /etc/mysql/mariadb.conf.d/50-server.cnf
systemctl start mariadb
mysql -u root -e \
  "CREATE DATABASE IF NOT EXISTS owncloud; \
  CREATE USER IF NOT EXISTS 'owncloud'@'localhost' IDENTIFIED BY 'password'; \
  GRANT ALL PRIVILEGES ON *.* TO 'owncloud'@'localhost' WITH GRANT OPTION; \
  FLUSH PRIVILEGES;"
```



It is recommended to run the mysqltuner script to analyze the database configuration after running with load for several days.

Once the database installation is complete, refer to the Database section in the Hardening and Security Guidance guide for additional important information.

##Enable the Recommended Apache Modules

```bash
a2enmod dir env headers mime rewrite setenvif
systemctl restart apache2
```

#Installation

##Download ownCloud

```bash
cd /var/www/
wget https://download.owncloud.com/server/stable/owncloud-complete-latest.tar.bz2 && \
tar -xjf owncloud-complete-latest.tar.bz2 && \
chown -R www-data. owncloud
```
##Install ownCloud

```bash
occ maintenance:install \
    --database "mysql" \
    --database-name "owncloud" \
    --database-user "owncloud" \
    --database-pass "password" \
    --data-dir "/var/www/owncloud/data" \
    --admin-user "admin" \
    --admin-pass "admin"
```
##Configure ownCloud’s Trusted Domains

```bash
my_ip=$(hostname -I|cut -f1 -d ' ')
occ config:system:set trusted_domains 1 --value="$my_ip"
occ config:system:set trusted_domains 2 --value="$my_domain"
```

##Configure the cron Jobs

Set your background job mode to cron:

```bash
occ background:cron
```

Set the execution of the cron job to every 15 minutes and the cleanup of chunks every night at 2 am:

```bash
echo "*/15  *  *  *  * /var/www/owncloud/occ system:cron" \
  | sudo -u www-data -g crontab tee -a \
  /var/spool/cron/crontabs/www-data
echo "0  2  *  *  * /var/www/owncloud/occ dav:cleanup-chunks" \
  | sudo -u www-data -g crontab tee -a \
  /var/spool/cron/crontabs/www-data
```
If you need to sync your users from an LDAP or Active Directory Server, add this additional cron job. Every 6 hours this cron job will sync LDAP users in ownCloud and disable the ones who are not available for ownCloud. Additionally, you get a log file in /var/log/ldap-sync/user-sync.log for debugging.

```bash
echo "1 */6 * * * /var/www/owncloud/occ user:sync \
  'OCA\User_LDAP\User_Proxy' -m disable -vvv >> \
  /var/log/ldap-sync/user-sync.log 2>&1" \
  | sudo -u www-data -g crontab tee -a \
  /var/spool/cron/crontabs/www-data
mkdir -p /var/log/ldap-sync
touch /var/log/ldap-sync/user-sync.log
chown www-data. /var/log/ldap-sync/user-sync.log
```

##Configure Caching and File Locking

```bash
occ config:system:set \
   memcache.local \
   --value '\OC\Memcache\APCu'
occ config:system:set \
   memcache.locking \
   --value '\OC\Memcache\Redis'
occ config:system:set \
   redis \
   --value '{"host": "127.0.0.1", "port": "6379"}' \
   --type json
```

##Configure Log Rotation

```bash
FILE="/etc/logrotate.d/owncloud"
sudo cat <<EOM >$FILE
/var/www/owncloud/data/owncloud.log {
  size 10M
  rotate 12
  copytruncate
  missingok
  compress
  compresscmd /bin/gzip
}
EOM
```

##Finalize the Installation

Make sure the permissions are correct:

```bash
cd /var/www/
chown -R www-data. owncloud
```

ownCloud is now installed. You can confirm that it is ready to enable HTTPS (for example using Let’s Encrypt) by pointing your web browser to your ownCloud installation.

To check if you have installed the correct version of ownCloud and that the occ command is working, execute the following:

```bash
occ -V
echo "Your ownCloud is accessable under: "$my_ip
echo "Your ownCloud is accessable under: "$my_domain
echo "The Installation is complete."
```









