---
layout: blog
title: Installing and securing Wordpress from scratch
category: IT
tags: [wordpress, multisite, installation, lets encrypt, https, ssl]
description: How to install multiple wordpress and enable SSH and other options (WIP)
---

In the next article we cover multiple topics related with Wordpress. It could be used as a full guide or as a reference related to a specific subject. 

All technical instructions are specified for Debian 9.

Here is the index with all points:

- [Secure installation](#secure-installation)
- [Virtual hosts configuration](#virtual-hosts-configuration)
- [Let's Encrypt setup](#lets-encrypt-setup)
- [Enabling HTTPS in Wordpress](#enabling-https-in-wordpress)
- [Hardening Wordpress](#hardening-wordpress)
- [References](#references)

## Secure installation

We could find online thousands of tutorials covering this topic, but it's hard to find one where you don't install old (and unsecure) software.

Main problem here is PHP version hosted in repositories. Usually, the standard installation will be 5.6.x (maybe 7.0.x if you are lucky), which is a kind of lottery [in terms of security](https://www.cvedetails.com/version-list/74/128/6/PHP-PHP.html?sha=0bb72d05caaf4fdd8d5e6106ced1cadc8a6e1095&order=1&trc=1127). Unless you need a extremely specific version because of plugins you'll need for Wordpress, **you should install the latest version available**.

Here is a summary of all steps for a fresh installation, including PHP 7.3.

### Core packages
```
apt-get update
apt-get install apache2 mysql-server

# Minimal secure performance related to SQL
mysql_secure_installation
``` 

### PHP 7.3

If we perform a search like `apt-cache search php7.3`, probably we will get no results. In this case, we have to add a new repository to our sources list:

```
echo "deb http://mirrordirector.raspbian.org/raspbian/ buster main contrib non-free rpi" > /etc/apt/sources.list.d/10-buster.list
vi /etc/apt/preferences.d/10-buster
	
-----------
# File content
Package: *
Pin: release n=stretch
Pin-Priority: 900

Package: *
Pin: release n=buster
Pin-Priority: 750
-----------
	
apt-get update
apt install -y -t buster php7.3-fpm php7.3-curl php7.3-gd php7.3-intl php7.3-mbstring php7.3-mysql php7.3-imap php7.3-opcache php7.3-sqlite3 php7.3-xml php7.3-xmlrpc php7.3-zip php7.3-bcmath php-apcu libapache2-mod-php7.3
```

## Virtual hosts configuration

Apache - Virtualhost
	cd /etc/apache2/sites-available
	vi sitename.conf
	-----------
	<VirtualHost *:80>
	        ServerName domainname.com
	        ServerAlias www.domainname.com

	        ServerAdmin webmaster@localhost
	        DocumentRoot /var/www/html/path/folder

	        ErrorLog ${APACHE_LOG_DIR}/domainname-error.log
	        CustomLog ${APACHE_LOG_DIR}/domainname-access.log combined

	</VirtualHost>
	-----------
	a2ensite sitename.conf
	/etc/init.d/apache2 restart

MySQL
	CREATE DATABASE databasename;
	CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'user_password';
	GRANT ALL PRIVILEGES ON databasename.* TO "wordpressusername"@'localhost' IDENTIFIED BY "password";
	FLUSH PRIVILEGES;

Wordpress
	wget https://wordpress.org/latest.tar.gz
	tar -xzvf latest.tar.gz

### References
- PHP custom source
