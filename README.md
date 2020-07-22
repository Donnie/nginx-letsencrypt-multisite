# Linode/DO + Ubuntu 16.04 LTS + UFW + Nginx (multi-site) + MySQL + phpMyAdmin + PHP 7.2 + Let's Encrypt (A+ SSL) + Cloudflare + Wordpress

## Installations

### Set up SSH key on a Linode/DO

#### Linode / Digital Ocean

Setup a new Linode or a Digital Ocean Droplet with Ubuntu 16.04 LTS

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
1. SSH to your Linode VPS using your root password. You can use FileZilla or Putty or both
2. Create folder `/root/.ssh`

`mkdir ~/.ssh`

3. Change folder access permission

`chmod 0700 ~/.ssh`

4. Upload the "authorized_keys" file to `/root/.ssh`

5. Change file access permission

`chmod 0644 ~/.ssh/authorized_keys`

You should now be able to login using the SSH key

6. Edit your `/etc/ssh/sshd_config` file to change

`PasswordAuthentication yes` to `PasswordAuthentication no`

This would disable password based login and would allow only SSH access, so make sure you have a working SSH key before doing this.

7. Hit reboot `reboot`

### Ubuntu 16.04 LTS
First thing you should do after logging in via SSH is update

`sudo apt-get update`

if update is getting stuck then force IPv4, edit /etc/gai.conf and delete the pound (#) from this line
`#precedence ::ffff:0:0/96  100`

then set timezone

`timedatectl set-timezone "Asia/Kolkata"`

#### Install PPA repository for Nginx, Certbot & PHP 
`sudo apt-get install software-properties-common python-software-properties`

`sudo apt-get install python3-pyasn1`

`sudo add-apt-repository ppa:certbot/certbot`

`sudo apt-get install python-certbot-nginx`

`sudo add-apt-repository ppa:ondrej/php`

`sudo add-apt-repository ppa:ondrej/nginx`

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

PS: If you are using Amazon EC2, then make sure your security group allows `0.0.0.0/0` in HTTP and HTTPS

At this point you are good enough to serve static html files from `/var/www/html` webroot

### MySQL
Install MySQL

`sudo apt-get install mysql-server`

Take care to use a strong password and keep it saved somewhere else

Configure secure MySQL

`mysql_secure_installation`

### PHP 7
Install PHP FastCGI Process Manager, additional SQL Helper Package and with other plugins like curl, mcrypt, etc.

`sudo apt install php7.2 php7.2-bcmath php7.2-bz2 php7.2-cli php7.2-common php7.2-curl php7.2-fpm php7.2-gd php7.2-intl php7.2-json php7.2-mbstring php7.2-mysql php7.2-opcache php7.2-pspell php7.2-soap php7.2-tidy php7.2-xml php7.2-xmlrpc php7.2-xsl php7.2-zip`

disable fix_pathinfo for better security

`sudo sed -i s/\;cgi\.fix_pathinfo\s*\=\s*1/cgi.fix_pathinfo\=0/ /etc/php/7.2/fpm/php.ini`

Restart PHP

`sudo systemctl restart php7.2-fpm`

### phpMyAdmin
`sudo apt-get update`

`sudo apt-get install phpmyadmin`

When asked to choose between `apache` and `lighttpd` choose none and simply press TAB key to go forward.

The next prompt will ask if you would like `dbconfig-common` to configure a database for phpmyadmin to use. Select "Yes" to continue.

The next prompt shall ask you the password, provide the MySQL root password.

___

## Configurations
For the purpose of this configuration part let's assume you are setting up the VPS for the first time. And the domain you are setting up is `domain.ga`. You can substitute `domain.ga` with your own domain name.

### PHP FPM Config
Keep a backup of the existing www.conf file

`mv /etc/php/7.2/fpm/pool.d/www.conf{,.bak}`

Create a new file

`/etc/php/7.2/fpm/pool.d/domain.ga.conf`

and put the following in the file
```
[domain.ga]
user = www-data
group = www-data
listen = /run/domain.ga-fpm.sock
listen.owner = www-data
listen.group = www-data
pm = ondemand
pm.process_idle_timeout = 30s
pm.max_requests = 512
pm.max_children = 30
```

Issue a PHP restart:

`service php7.2-fpm restart`

### PHP Web root

Create the root folder for the domain (assuming domain.ga)

`mkdir -p /var/www/domain.ga/{files,log}`

Inside `files` add `index.php` with a line of code

`echo "<?php echo 'Hello World';" >> /var/www/domain.ga/files/index.php`

inside the `log` folder create blank files

`touch access.log error.log`

### Cloudflare
Add your domain name to Cloudflare and enable the DNS proxy

### Nginx Server Blocks
#### Nginx configuration snippets
write `/etc/nginx/sites-available/domain.ga`

```
server {
	listen 80;
	server_name domain.ga;
	root /var/www/domain.ga/files;
	index index.php index.html index.htm;

	access_log /var/www/domain.ga/log/access.log;
	error_log  /var/www/domain.ga/log/error.log notice;

	location ~ \.php$ {
		try_files $uri =404;
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		include fastcgi_params;
		fastcgi_read_timeout 300;
		fastcgi_intercept_errors on;
		fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
		fastcgi_pass unix:/run/domain.ga-fpm.sock;
	}
}
```

create a shortcut of the file from `/etc/nginx/sites-enabled`

`ln -s /etc/nginx/sites-available/domain.ga /etc/nginx/sites-enabled/`

Reload Nginx

`service nginx restart`

At this point you are ready to talk PHP on domain.ga, visit domain.ga to make sure you see the 'Hello World' message

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

#### Create a strong Diffie-Hellman parameter

`openssl dhparam -out /etc/nginx/dhparam.pem 4096`

This will take sometime, go and grab a coffee.
OR
You can use the dhparam.pem file available with this repo and put it in the nginx directory.

#### Append Nginx SSL Server Block
Open `/etc/nginx/sites-available/domain.ga`

And append the below lines after the first server block

```
server {
	listen 443 ssl;
	server_name domain.ga;
	root /var/www/domain.ga/files;
	index index.php index.html index.htm;

	access_log /var/www/domain.ga/log/access.log;
	error_log  /var/www/domain.ga/log/error.log notice;

	ssl_certificate /etc/letsencrypt/live/domain.ga/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/domain.ga/privkey.pem;

	location ~ \.php$ {
		try_files $uri =404;
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		include fastcgi_params;
		fastcgi_read_timeout 300;
		fastcgi_intercept_errors on;
		fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
		fastcgi_pass unix:/run/domain.ga-fpm.sock;
	}
}
```

open nginx.conf and replace the SSL section with this:

```
ssl_dhparam /etc/nginx/dhparam.pem; 
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
```

Check Nginx configurations

`nginx -t`

You should see this:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx

`nginx -s reload`

At this point you can access https://domain.ga and see `Hello World` on it.

You should also check if you are getting an A+ SSL rating
Visit https://www.ssllabs.com/ssltest/analyze.html?d=domain.ga after disabling DNS proxy on Cloudflare

#### Auto-renew using crontab:

`sudo crontab -e` 

Add the following lines to renew certs every 1 and 15th day of the month at 2:00am

`00 2 1 * * /usr/bin/certbot renew -q
00 2 15 * * /usr/bin/certbot renew -q`

### Wordpress (Optional)
#### Configure Nginx server block

Replace try_files directive with this

`try_files $uri $uri/ /index.php?q=$uri&$args;`

For additional security add this location block to deny access to php files

```
# SECURITY : Deny all attempts to access PHP Files in the uploads directory
location ~* /(?:uploads|files)/.*\.php$ {
	deny all;
}
location ~ /\.ht {
	deny all;
}
```

This one is useful for sitemap links

```
# PLUGINS : Enable Rewrite Rules for Yoast SEO SiteMap
rewrite ^/sitemap_index\.xml$ /index.php?sitemap=1 last;
rewrite ^/([^/]+?)-sitemap([0-9]+)?\.xml$ /index.php?sitemap=$1&sitemap_n=$2 last;
```

#### Setup MySQL Database with User
Login to MySQL

`mysql -u root -p`

Create user

`CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';`

Create new database

`CREATE DATABASE domain_blog;`

Grant privileges

`GRANT ALL PRIVILEGES ON domain_blog. * TO 'newuser'@'localhost' IDENTIFIED BY 'password';`

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

`mv -a /var/www/domain.ga/files/wordpress/* /var/www/domain.ga/files/`

adjust file ownership settings

`sudo chown -R www-data:www-data /var/www/domain.ga/files/`

set file group inheritance

`sudo find /var/www/domain.ga/files/ -type d -exec chmod g+s {} \;`

At this point you can visit your domain https://domain.ga and finish the Wordpress installation with the correct database credentials

### Adding new sites
1. Create folder in /var/www/ with files and log directory
2. Add PHP-FPM config file into /etc/php/7.2/fpm/pool.d for the new site
3. Add the server block to /etc/nginx/sites-available and create shortcut of it on sites-enabled
4. Restart PHP
5. Restart Nginx
6. Get an SSL certificate from Let's Encrypt
7. Add SSL server block to server configuration file in /etc/nginx/sites-available/
8. Restart Nginx
9. Add site to cloudflare
10. Edit wp-config.php file and connect with database
11. rename Facebook plugin folder if there is an error with Wordpress, it's known to create troubles while shifting hosts

