---
layout: blog
title: Installing and securing Wordpress from scratch
category: IT
tags: [wordpress, multisite, installation, lets encrypt, https, ssl]
excerpt_separator: <!--more-->
---

In the next article we cover multiple topics related with Wordpress. It could be used as a full guide or as a reference related to a specific subject. 

All technical instructions are specified for Debian 9 (stretch).

***Date of last update*** *December 6th, 2019*
<!--more-->
Here is the index with all points:

- [Secure installation](#secure-installation)
- [Virtual hosts configuration](#virtual-hosts-configuration)
- [Let's Encrypt setup](#lets-encrypt-setup)
- [Enabling HTTPS in Wordpress](#enabling-https-in-wordpress)
- [Hardening Wordpress](#hardening-wordpress)
- [Multiple Wordpress with different domains](#multiple-Wordpress-with-different-domains)
- [Backup](#backup)
- [References](#references)

## Secure installation

We could find online thousands of tutorials covering this topic, but it's hard to find one where you don't install old (and unsecure) software.

Main problem here is PHP version hosted in repositories. Usually, the standard installation will be 5.6.x (maybe 7.0.x if you are lucky), which is a kind of lottery [in terms of security](https://www.cvedetails.com/version-list/74/128/6/PHP-PHP.html?sha=0bb72d05caaf4fdd8d5e6106ced1cadc8a6e1095&order=1&trc=1127). Unless you need a extremely specific version because of plugins you'll need for Wordpress, **you should install the latest version available**.

Here is a summary of all steps for a fresh installation, including PHP 7.3.

#### Core packages

{% highlight bash %}apt-get update
apt-get install apache2 mysql-server

# Remove apache2 info file
rm -f /var/www/html/index.html
# Minimal secure performance related to SQL
mysql_secure_installation{% endhighlight %} 

#### Preparing MySQL database

Although this step could be considered as part of hardening, it's mandatory to perform it in the installation phase, due of the final Wordpress configuration.

First, we need to create a new database and a new user with privileges: 

{% highlight sql %}CREATE DATABASE <databasename>;
CREATE USER '<newuser>'@'localhost' IDENTIFIED BY '<user_password>';
GRANT ALL PRIVILEGES ON <databasename>.* \
TO "<wordpressusername>"@'localhost' \
IDENTIFIED BY "<user_password>";
FLUSH PRIVILEGES;
{% endhighlight %}

I suggest to use mnemonic names related to the blog but, at the same time, hard to guess by an attacker. For example, if our website is `https://www.cellphonereviews.com`, one customization could be `cprevs_db` (database) and `adm_cpr` (username).

#### PHP 7.3

If we perform a search like `apt-cache search php7.3`, probably we will get no results. In this case, we have to add a new repository to our sources list:

{% highlight bash %}apt -y install lsb-release apt-transport-https ca-certificates 
cd /etc/apt/trusted.gpg.d
wget -O php.gpg https://packages.sury.org/php/apt.gpg
cd /etc/apt/sources.list.d
echo "deb https://packages.sury.org/php/ stretch main" > php7.list{% endhighlight %}

Now install the last version of PHP hosted in the added repository:

{% highlight bash %}apt-get update
apt install -y php7.3-fpm php7.3-curl php7.3-gd php7.3-intl \
php7.3-mbstring php7.3-mysql php7.3-imap php7.3-opcache php7.3-sqlite3 \
php7.3-xml php7.3-xmlrpc php7.3-zip php7.3-bcmath php-apcu \
libapache2-mod-php7.3{% endhighlight %}

Check the version is the expected one with `php -v`.

#### Wordpress

Download with `wget` or `curl` the last version of Wordpress:

{% highlight bash %}cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
mv wordpress/ /var/www/html
chown -R www-data:www-data /var/www/html{% endhighlight %}

Now you can open a browser, navigate to `http://<your_ip_address>/wordpress` and perform the final installation with Wordpress' wizard. 

As we saw in the previous step [Preparing MySQL database](#preparing-mysql-database), here is another hardening chance to secure our installation. When the Wordpress' wizard will ask for a database prefix, instead of leave `wp_`, we could set whatever we want. Related to the previous example (`https://www.cellphonereviews.com`), some options could be `cprwp_` or `rpc570_`.

## Virtual hosts configuration

Fresh Apache2 installations includes a `000-default.conf` site enabled, allowing HTTP access to files under `/var/www/html`. But, if we store all Wordpress files on a subfolder or other path, we could get some display issues (or even disclose information of our system).

On [Let's Encrypt setup](#lets-encrypt-setup) step we will get a brand-new configuration site. For now, we need an initial configuration which will be upgraded later. We can edit `000-default.conf` or create a new one (if you choose this option, don't forget to enable with `a2ensite`!):

Our starting configuration will be the next one:

{% highlight bash %}cd /etc/apache2/sites-available
vi sitename.conf
# Beginning of sitename.conf file
<VirtualHost *:80>
    ServerName <MYWEBSITE>
    ServerAlias www.<MYWEBSITE>
    ServerAdmin admin@<MYWEBSITE>
    DocumentRoot /var/www/html/<MYWEBSITE>

    <Directory /var/www/html/<MYWEBSITE>/>
        Options FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory /var/www/html/<MYWEBSITE>/>
        RewriteEngine on
        RewriteBase /
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*) index.php [PT,L]
    </Directory>
</VirtualHost>
# End of sitename.conf file

a2ensite sitename.conf
systemctl reload apache2{% endhighlight %}

## Lets Encrypt setup

In order to enable HTTPS into Wordpress, one of the requirements (if we don't want to get a "THIS CONNECTION IS NOT SECURE" window message) is to have installed a certificate signed by a trusted Certification Authority (CA). If we are not a big company, the best and cheapest option is use one provided by **Let's Encrypt** via CertBot. 

Due of development of OS and CertBot, installation instructions and requirements can be slighty different (for example, all procedure change between Debian 8 and Debian 9). Remember, __this article is based on Debian 9__. If you need other distribution or HTTP server, [check this link](https://certbot.eff.org/instructions) to see all alternatives available.

### CertBot installation and configuration

Before start, we need to enable stretch backports repository. To achieve this step, open `/etc/apt/sources.list` and add the next line at the end:

`deb http://deb.debian.org/debian stretch-backports main`

Now we can install CertBot from backports repository:

`apt-get install certbot python-certbot-apache -t stretch-backports`

After installing Certbot, create a file for Let’s Encrypt to validate our domain in the `${webroot-path}/.well-known/acme-challenge directory`.

To do that, create the directory and give Apache2 access to it:

```
mkdir -p /var/lib/letsencrypt/.well-known
chgrp www-data /var/lib/letsencrypt
chmod g+s /var/lib/letsencrypt
```

Next, create a well-known challenge file and enable it with `a2enconf`:
{% highlight bash %}
nano /etc/apache2/conf-available/well-known.conf
# Content of the file
Alias /.well-known/acme-challenge/ "/var/lib/letsencrypt/.well-known/acme-challenge/"
<Directory "/var/lib/letsencrypt/">
    AllowOverride None
    Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
    Require method GET POST OPTIONS
</Directory>
# Enable the config in apache2 (don't forget to restart)
a2enconf well-known.conf
{% endhighlight %}

At this point, your domain should be configured with your server IP address (which is something not covered in this tutorial). Before we request the certificate, we'll check our apache2 configuration to see all mods are enabled with the next commands:

```
a2enmod ssl headers http2
a2enconf well-known
```

Now we could execute certbot to perform some checks and get our certs:

`certbot certonly --agree-tos --email admin@example.com --webroot -w /var/lib/letsencrypt/ -d example.com -d www.example.com`

If everything works fine, you will get a message like:

{% highlight bash %}
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/www.fakedomain.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/www.fakedomain.com/privkey.pem
   Your cert will expire on 2019-10-24. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
{% endhighlight %}

Now it's time to generate a Diffie–Hellman key exchange (DH) certificate to securely exchange cryptographic keys. To do that, run the commands below to generate a certificate with 2048 bit:

`openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048`

When it finish (it could take some time), we will update our previous apache configuration file created on step [Virtual host configuration](#virtual-host-configuration). The new one should look like the next one:

{% highlight bash %}
<VirtualHost *:80>
  ServerName fakedomain.com
  ServerAlias www.fakedomain.com

  Redirect permanent / https://fakedomain.com/
</VirtualHost>

<VirtualHost *:443>
  ServerName fakedomain.com
  ServerAlias www.fakedomain.com
  DocumentRoot /var/www/html/esplur

  Protocols h2 http:/1.1

  ErrorLog ${APACHE_LOG_DIR}/fakedomain.com-error.log
  CustomLog ${APACHE_LOG_DIR}/fakedomain.com-access.log combined

  SSLEngine On
  SSLCertificateFile /etc/letsencrypt/live/fakedomain.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/fakedomain.com/privkey.pem
  SSLOpenSSLConfCmd DHParameters "/etc/ssl/certs/dhparam.pem"

  SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
  SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
  SSLCompression off
  SSLUseStapling on

  <Directory /var/www/html/fakedomain.com/>
       Options FollowSymlinks
       AllowOverride All
       Require all granted
  </Directory>

  <Directory /var/www/html/fakedomain.com/>
       RewriteEngine on
       RewriteBase /
       RewriteCond %{REQUEST_FILENAME} !-f
       RewriteRule ^(.*) index.php [PT,L]
  </Directory>
</VirtualHost>
{% endhighlight %}

Next you will need to configure a server cache for the OCSP status information. This will be done in the Apache SSL config file:

```
nano /etc/apache2/mods-available/ssl.conf
# Set the location of the SSL OCSP Stapling Cache
SSLStaplingCache shmcb:/tmp/stapling_cache(128000)
```

Our final step is to check the auto-renew command, to see if we have to do a troubleshooting and avoid to expire our certificate. Execute the following command to test:

`certbot renew --dry-run`

You can add it to crontab to automatically update:

`30 2 * * * /usr/bin/certbot renew & > /dev/null`

## Enabling HTTPS in Wordpress

In the administration panel, under `Settings -> General`, we need to edit the URL of the blog. Just change the `http` header into `https` and apply changes.

Usually other docs and sites (like StackOverflow) talk about editing `.htaccess`. Although I didn't need it on my installation, here is the snippet with the code in case you need it:

{% highlight bash %}<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{SERVER_PORT} !^443$
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
    RewriteBase /
    RewriteRule ^index\.php$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule . /index.php [L]
</IfModule>{% endhighlight %}

This code (excluding `IfModule` XML tags) can be added in the virtual host file instead of `.htaccess`.

**Note:** if you are going to change Worpress configuration to use a non-default permalink option, please take a look at our [issues](#issues) section.

## Hardening Wordpress

- https://mythemeshop.com/blog/custom-wordpress-login-url/
- Change files and directories permissions
- https://make.wordpress.org/hosting/handbook/handbook/security/

## Multiple Wordpress with different domains

Although Wordpress Codex documentation covers a lot of topics, sometimes it's hard to find (or even it doesn't exist) the information we require for our use case. One example is how to get multiple Wordpress with different domains and different installations (it seems they focus on multisite option with one WP installation, named as [Network](https://wordpress.org/support/article/create-a-network/). 

Here you will a custom example where we follow all previous steps to install two Wordpress with **different domains**, `www.myportfolio.com` and `www.myhobbies.com`, and focusing only on the code we need to customize and explain:

### [1. Secure installation](#secure-installation)

When we configure MySQL, we should create a new database **AND** user, to isolate websites.

{% highlight sql %}# www.myportfolio.com
CREATE DATABASE portfoliodb;
CREATE USER 'portfoliosql'@'localhost' \
IDENTIFIED BY 'PleaseRemoveDolphinFromTop1000UsedPasswords!!!';
GRANT ALL PRIVILEGES ON portfoliodb.* TO "portfoliosql"@'localhost' \
IDENTIFIED BY "PleaseRemoveDolphinFromTop1000UsedPasswords!!!";
# www.myhobbies.com
CREATE DATABASE hobbiesdb;
CREATE USER 'hobbiessql'@'localhost' \
IDENTIFIED BY 'PleaseDontUseTheSamePasswordAsThePrevious1!!!';
GRANT ALL PRIVILEGES ON hobbiesdb.* TO "hobbiessql"@'localhost' \
IDENTIFIED BY "PleaseDontUseTheSamePasswordAsThePrevious1!!!";
# Update all changes
FLUSH PRIVILEGES;{% endhighlight %}
    
After untar zip Wordpress file, instead of move the folder, we could copy it and rename it for each website. For example:
    
{% highlight bash %}cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
cp -r wordpress/ /var/www/html/myportfolio
cp -r wordpress/ /var/www/html/myhobbies  
chown -R www-data:www-data /var/www/html{% endhighlight %}
    
### [2. Virtual hosts configuration](#virtual-hosts-configuration)
Every blog must have an initial virtual host configuration, specially if we want to make CertBot work. To achieve this task, we need to copy and modify the configuration file previously seen under [CertBot installation and configuration](#certBot-installation-and-configuration), without forgetting the next points:
1. Modify all variables related to new site (ServerName, ServerAlias, DocumentRoot paths...)
2. Enable the site with `a2ensite`
3. Restart Apache

### [3. Let's Encrypt setup](#lets-encrypt-setup)
```
certbot certonly --agree-tos --email admin@example.com --webroot -w /var/lib/letsencrypt/ -d myportfolio.com -d www.myportfolio.com
certbot certonly --agree-tos --email admin@example.com --webroot -w /var/lib/letsencrypt/ -d myhobbies.com -d www.myhobbies.com
```

### [4. Enabling HTTPS in Wordpress](#enabling-https-in-wordpress)
As showed before.

## Backup

Here is a little script I've customized from the original one found in [James Rascal's repository](https://github.com/jamesrascal/wordpress-backup).

Although here is the code, you could find the latest version [here](https://github.com/dferrero/wordpress-backup).

{% highlight bash %}
#!/bin/bash
WEBPATH=/var/www/html
BACKUP_LOCATION=/home/backups
KEEPDAYS=8

for webdir in `find $WEBPATH -maxdepth 1 -mindepth 1 -type d`; do
    currentdir=`echo $webdir | cut -d'/' -f5`
    echo "Starting backup from $WEBPATH/$currentdir"
    wp_config=$webdir/wp-config.php
    # BackupName Date and time
    backupname=$(date +%y%m%d)-$currentdir

    # Creates a Backup Directory if one does not exist.
    mkdir -p ${BACKUP_LOCATION}/${currentdir}/

    # Make Backup location the current directory
    cd ${BACKUP_LOCATION}/${currentdir}

    # Verifing wp_config exists
    if [ -f "$wp_config" ]; then
        # Pulls Database info from WP-config
        db_name=$(grep DB_NAME "${wp_config}" | cut -f4 -d"'")
        db_user=$(grep DB_USER "${wp_config}" | cut -f4 -d"'")
        db_pass=$(grep DB_PASSWORD "${wp_config}" | cut -f4 -d"'")
        table_prefix=$(grep table_prefix "${wp_config}" | cut -f2 -d"'")

        # MySQL Takes a Dump and compress the Home Directory
        mysqldump -u ${db_user} -p${db_pass} ${db_name} | gzip > ./${backupname}-DB.sql.gz
        tar zcPf ./${backupname}-FILES.tar.gz ${webdir}

        # Compresses the MySQL Dump and the Home Directory
        tar zcPf ./${backupname}.tar.gz ./${backupname}-FILES.tar.gz ./${backupname}-DB.sql.gz
        chmod 600 ./${backupname}.tar.gz

        # Generates the Backup Size
        #FILENAME=${BACKUP_LOCATION}/${currentdir}/${backupname}.tar.gz
        #FILESIZE=$(du -h "$FILENAME")

        #Removes the SQL dump and Home DIR to conserve space
        rm -rf ./${backupname}-FILES.tar.gz ./${backupname}-DB.sql.gz
    else
        echo "No wp-config.php found; saving full folder";
        tar zcPf ./${backupname}.tar.gz ${webdir}
    fi

    #Deletes any Backup older than X days
    find ${BACKUP_LOCATION}/${currentdir}/ -type f -mtime +${KEEPDAYS} -exec rm {} \;
done
{% endhighlight %}

## Issues

Here is a recopilation of usual erros I've found during/after a Wordpress installation.

_0x00 - Let's Encrypt repository is not signed_

If you are getting an error message related to a repository which is not signed after adding it to `/etc/apt/sources.list` and executing `apt-get update`, probably your distribution is not a Debian 9 Stretch. You will have to look for the repository which is related to your OS and use it instead of the suggested one. I saw this problem when I tried to install on a Raspberry Pi 4.

_0x01 - Wordpress is not sending emails_

This issue is still being reviewed on my own instance. If you need a starting point to begin to deal with, here is a [link](https://kinsta.com/knowledgebase/wordpress-not-sending-emails/)

## References
- [Wordpress Codex](https://codex.wordpress.org/)
- [Install PHP 7.3 on Debian](https://computingforgeeks.com/how-to-install-php-7-3-on-debian-9-debian-8/)
- [How to install Wordpress with Apache2 and Let's Encrypt](https://websiteforstudents.com/how-to-install-wordpress-with-apache2-and-lets-encrypt-ssl-tls-certificates-on-ubuntu-16-04-18-04/)
- [Wordpress Permalinks are not working](https://stackoverflow.com/questions/19400749/wordpress-permalinks-not-working-htaccess-seems-ok-but-getting-404-error-on-pa)
