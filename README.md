# Linode + Ubuntu 16.04 LTS + UFW + Nginx (multi-site) + MySQL + phpMyAdmin + PHP 7 + Let's Encrypt (A+ SSL) + Cloudflare + Wordpress

## Installations

### Set up SSH key on a Linode Box

#### Linode

Setup a new Linode with Ubuntu 16.04 LTS

#### Create SSH key
1. Download PuttyGen
2. Start PuTTYgen
3. Select SSH-2 RSA from the bottom menu of Type of key to generate
4. Keep number of bits: 4096
5. Click Generate button and move your mouse pointer randomly to generate random values
6. Once the process is complete you would see the public key 
7. Put a key name in the key comment field
8. Save the public key to a file named key.pub
9. Save the private key to a file named key.ppk
10. Right-click on the text label showing the pub key value and save it as "authorized_keys"

#### Configure SSH key
1. SSH to your Linode VPS using your root password. You can use FileZilla or Putty both
2. Create `/root/.ssh`

`mkdir ~/.ssh`

3. Change folder access permission

`chmod 0700 ~/.ssh`

4. Upload the "authorized_keys" file to `/root/.ssh`

5. Change file access permission

`chmod 0644 ~/.ssh/authorized_keys`

You should now be able to login using the SSH key

6. Edit your `/etc/ssh/sshd_config` file to change

`PasswordAuthentication yes` to `PasswordAuthentication no`

7. Hit reboot `reboot`

### Ubuntu 16.04 LTS
First thing you should do after logging in via SSH is update

`sudo apt-get update`

### Ubuntu Firewall
Allow http, https and ssh access only

`sudo ufw allow http`

`sudo ufw allow https`

`sudo ufw allow ssh`

`sudo ufw enable`

You will get this response

`Command may disrupt existing ssh connections. Proceed with operation (y|n)?` 

reply with `y`

Finally check status

`sudo ufw status`

You should get this response

```
Status: active

To						 Action	  From
--						 ------	  ----
80						 ALLOW	   Anywhere
443						ALLOW	   Anywhere
22						 ALLOW	   Anywhere
80 (v6)					ALLOW	   Anywhere (v6)
443 (v6)				   ALLOW	   Anywhere (v6)
22 (v6)					ALLOW	   Anywhere (v6)
```

### Nginx
Now install Nginx

`sudo apt-get install nginx`

Once you are done, you can visit your IP Address on your browser and it would show:

```
Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```

At this point you are good enough to serve static html files 

### MySQL
Install MySQL

`sudo apt-get install mysql-server`

Configure secure MySQL

`mysql_secure_installation`

### PHP 7
Install PHP FastCGI Process Manager and additional SQL Helper Package

`sudo apt-get install php-fpm php-mysql`

Edit PHP INI file.

You can either browse through FileZilla `/etc/php/7.0/fpm/php.ini`

or use Putty with this command `sudo nano /etc/php/7.0/fpm/php.ini` 

and set `cgi.fix_pathinfo=0`

Restart PHP

`sudo systemctl restart php7.0-fpm`

### phpMyAdmin
`sudo apt-get update`

`sudo apt-get install phpmyadmin`

When asked to choose between `apache` and `lighttpd` choose none and simply press TAB key to go forward.

The next prompt will ask if you would like `dbconfig-common` to configure a database for phpmyadmin to use. Select "Yes" to continue.

The next prompt shall ask you for a password provide the root password for MySQL.

### Install Let's Encrypt (certbot) for SSL
On Ubuntu systems, the Certbot team maintains a PPA. Once you add it to your list of repositories all you'll need to do is apt-get the following packages.

```
sudo apt-get update
sudo apt-get install software-properties-common python-software-properties
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

___

## Configurations
### PHP Web root

Create the root folder for the domain

`mkdir /var/www/domain.ga/files`

and add `index.php` with a line of code

`<?php echo 'Hello World';`

create the log folder

`mkdir /var/www/domain.ga/log`

create blank `access.log` and `error.log` files in the log folder

### Cloudflare
Add your domain name to Cloudflare and enable the DNS proxy

### Nginx Server Blocks
#### Nginx configuration snippets

make a `/etc/nginx/global` folder

put `common.conf` with this text

```
listen 80;

location / {
	try_files $uri $uri/ =404;
}

index index.php index.html index.htm index.nginx-debian.html;

location ~ \.php$ {
	include /etc/nginx/snippets/fastcgi-php.conf;
	fastcgi_pass unix:/run/php/php7.0-fpm.sock;
}

location ~ /\.ht {
	deny all;
}
```

and `wordpress.conf` with this

```
listen 80;

index index.php index.html index.htm index.nginx-debian.html;

location ~ \.php$ {
	include /etc/nginx/snippets/fastcgi-php.conf;
	fastcgi_pass unix:/run/php/php7.0-fpm.sock;
}

location ~ /\.ht {
	deny all;
}

location / {
	try_files $uri $uri/ /index.php?q=$uri&$args;
}

# SECURITY : Deny all attempts to access PHP Files in the uploads directory
location ~* /(?:uploads|files)/.*\.php$ {
	deny all;
}

# PLUGINS : Enable Rewrite Rules for Yoast SEO SiteMap
rewrite ^/sitemap_index\.xml$ /index.php?sitemap=1 last;
rewrite ^/([^/]+?)-sitemap([0-9]+)?\.xml$ /index.php?sitemap=$1&sitemap_n=$2 last;
```

#### Enable Sites

write `/etc/nginx/sites-available/domain.ga`

```
server {
	root /var/www/domain.ga/files;
		server_name domain.ga;
		access_log /var/www/domain.ga/log/access.log;
		error_log /var/www/domain.ga/log/error.log;
		include /etc/nginx/global/common.conf;
}
```

create a shortcut of the file from `/etc/nginx/sites-enabled`

`ln -s /etc/nginx/sites-available/domain.ga /etc/nginx/sites-enabled/`

Reload Nginx

`service nginx restart`

At this point you are ready to talk PHP on domain.ga

### phpMyAdmin web link

create a shortcut of phpMyAdmin in the root folder of your domain

`ln -s /usr/share/phpmyadmin/ /var/www/domain.ga/files`

Now you can access phpMyAdmin on domain.ga/phpmyadmin/

### Let's Encrypt
#### Let's Encrypt Certificate installation
Install certificates by --webroot plugin

```
sudo certbot certonly --webroot -w /var/www/domain.ga/files/ -d domain.ga
```

#### Create a strong Diffie-Hellman pair

`openssl dhparam -out /etc/letsencrypt/live/domain.ga/dhparam.pem 4096`

#### Append Nginx SSL Server Block
Open `/etc/nginx/sites-available/domain.ga`

And put the below server block at the end of the file

```
server {
	listen 443 ssl;
		root /var/www/domain.ga/files;
		server_name domain.ga;
		access_log /var/www/domain.ga/log/access.log;
		error_log /var/www/domain.ga/log/error.log;
		include /etc/nginx/global/common.conf;
	
	ssl on;
	ssl_certificate /etc/letsencrypt/live/domain.ga/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/domain.ga/privkey.pem;
	ssl_dhparam /etc/letsencrypt/live/domain.ga/dhparam.pem; 
}
```

open nginx.conf and replace the SSL section with this:

```
ssl_protocols TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
ssl_ecdh_curve secp384r1; 
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off; 
ssl_stapling on; 
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s; 
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header X-Robots-Tag none; 
```

Check Nginx configurations

`nginx -t`

You should see this:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx

`service nginx restart`

At this point you can access https://domain.ga and run PHP on it.

You should also check if you are getting an A+ SSL rating on https://www.ssllabs.com/ssltest/analyze.html?d=domain.ga

#### Auto-renew using crontab:

`sudo crontab -e` 

Add the following lines to renew certs every 1 and 15th day of the month at 2:00am

`00 2 1 * * /usr/bin/certbot renew -q`

`00 2 15 * * /usr/bin/certbot renew -q`

### Wordpress
#### Setup MySQL Database with User
Login to MySQL

`mysql -u root -p`

Create user

`CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';`

Create new database

`CREATE DATABASE domain_blog;`

Grant privileges

`GRANT ALL PRIVILEGES ON domain_blog. * TO 'newuser'@'localhost';`

Reload privileges

`FLUSH PRIVILEGES;`

exit mysql

`exit`

#### Get latest Wordpress version

move to your files directory

`cd /var/www/domain.ga/files`

download the latest version of Wordpress

`wget "https://wordpress.org/latest.zip";`

Unzip it

`unzip latest.zip`

and move the contents to the files directory

`mv -v wordpress/* /var/www/domain.ga/files/`
