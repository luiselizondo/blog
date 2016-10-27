---
layout: post
title: Ubuntu 12.04 + Nginx + Drupal
date: '2012-05-19 06:09:18'
---

No introductions, open terminal and ssh to your server. Only one consideration, I’m doing this on a AWS EC2 Instance.

<h3>Install packages</h3>
<code>sudo apt-get install nginx php5-fpm php5 mysql-common mysql-client-5.5 php5-mysql php5-common curl libcurl3 libcurl3-dev php5-curl php-pear make php5-gd postfix php5-memcache php-apc portmap nfs-common</code>
 
Note:
 I’m installing a NFS client on this server, if you don’t want to do this, remove portmap and nfs-common from the previous line. Also, we won’t be installing MySQL Server here, I’m running MySQL on a different box.

<h3>Install Drush</h3>

<code>
sudo pear channel-discover pear.drush.org ; sudo pear install drush/drush
sudo drush
sudo chown -R ubuntu:ubuntu /home/ubuntu/.drush
sudo chmod -R 770 /home/ubuntu/.drush
</code>

<h3>Backup original nginx configuration</h3>
<code>sudo cp nginx.conf nginx.conf.original</code>

<h3>Edit nginx configuration</h3>
<code>sudo nano /etc/nginx/nginx.conf</code>

Remove everything and add the following:
<code>
user www-data;
worker_processes 4;
pid /var/run/nginx.pid;

events {
worker_connections 768;
# multi_accept on;
}

http {

##
# Basic Settings
##

sendfile on;
tcp_nopush on;
tcp_nodelay on;
keepalive_timeout 65;
types_hash_max_size 2048;
# server_tokens off;

# server_names_hash_bucket_size 64;
# server_name_in_redirect off;

include /etc/nginx/mime.types;
default_type application/octet-stream;

##
# Logging Settings
##

access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;

##
# Gzip Settings
##

gzip on;
gzip_disable “msie6”;

# gzip_vary on;
gzip_proxied any;
gzip_comp_level 1;
# gzip_buffers 16 8k;
# gzip_http_version 1.1;
gzip_types
 text/plain text/css application/json application/x-javascript text/xml 
application/xml application/xml+rss text/javascript;

##
# nginx-naxsi config
##
# Uncomment it if you installed nginx-naxsi
##

#include /etc/nginx/naxsi_core.rules;

##
# nginx-passenger config
##
# Uncomment it if you installed nginx-passenger
##

#passenger_root /usr;
#passenger_ruby /usr/bin/ruby;

##
# Virtual Host Configs
##

include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;

#mail {
# # See sample authentication script at:
# # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
# # auth_http localhost/auth.php;
# # pop3_capabilities “TOP” “USER”;
# # imap_capabilities “IMAP4rev1” “UIDPLUS”;
#
# server {
# listen localhost:110;
# protocol pop3;
# proxy on;
# }
#
# server {
# listen localhost:143;
# protocol imap;
# proxy on;
# }
#}

}
</code>

Edit /etc/nginx/sites-available/defaultRemove everything and add the following

<code>
server {
server_name localhost;
root /var/www; ## <— Drupal path
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;

location = /favicon.ico {
log_not_found off;
access_log off;
}

location = /robots.txt {
allow all;
log_not_found off;
access_log off;
}

# This matters if you use drush
location = /backup {
deny all;
}

# Very rarely should these ever be accessed outside of your lan
location ~* .(txt|log)$ {
allow 192.168.0.0/16;
deny all;
}

location ~ ..*/.*.php$ {
return 403;
}

location / {
# This is cool because no php is touched for static content
try_files $uri @rewrite;
}

location @rewrite {
# Some modules enforce no slash (/) at the end of the URL
# Else this rewrite block wouldn’t be needed (GlobalRedirect)
rewrite ^/(.*)$ /index.php?q=$1;
}

location ~ .php$ {
fastcgi_split_path_info ^(.+.php)(/.+)$;
#NOTE: You should have “cgi.fix_pathinfo = 0;” in php.ini
include fastcgi_params;
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
fastcgi_pass 127.0.0.1:9000;
}

# Fighting with ImageCache? This little gem is amazing.
location ~ ^/sites/.*/files/styles/ {
try_files $uri @rewrite;
}

location ~* .(js|css|png|jpg|jpeg|gif|ico)$ {
expires max;
log_not_found off;
}
}</code>

This is for NFS, skip if you want

Edit /etc/idmapd.conf
<code>
sudo nano /etc/idmapd.conf
</code>

Change values to:

<code>
Nobody-User = ubuntu
Nobody-Group = root
</code>

<h3>Connect NFS</h3>
sudo mkdir /var/myfiles

Edit /etc/fstab and add the following line
<code>
ipgoeshere:/ /var/myfiles nfs4 _netdev,auto 0 0Mount
sudo mount -aCheck that everything is working by visitingcd /var/myfiles
</code>

<h3>Install PHP extensions</h3>
<code>sudo pecl install mongo uploadprogress</code>

Don’t forget to add the configuration files at /etc/php5/conf.d by creating mongo.ini and uploadprogress.inifiles. If you don’t know what you’re doing check one of the files in that directory.

<h3>Download site</h3>

<code>
cd /tmp
drush dl
sudo mv yourdrupaldirectory /var/www
</code>

<h3>Restart services</h3>

<code>
sudo service nginx restart
sudo service php5-fpm restart
</code>

Visit your site! It should be working and you should view the install screen! If not, google any errors, visit your log, check your DNS, remember that MySQL is not running on this box.