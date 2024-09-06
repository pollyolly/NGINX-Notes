# NGINX-ADMINISTRATION

### Install
```vim
sudo apt update
sudo apt install nginx
sudo apt install php7.4-fpm
```
### Adjust firewall
```vim
sudo ufw app list
sudo ufw allow 'Nginx Full' or sudo ufw allow 'Nginx HTTP' or sudo ufw allow 'Nginx HTTPS'
sudo ufw status
```
### Check status
```vim
service nginx status
```
### Test Nginx
```vim
curl -4 icanhazip.com
```
### Commands
```vim
$nginx -t (test nginx configuration if correct)
$service nginx start|restart|reload|stop|status
$service php7.4-fpm status 
```
### Setup Server block
```vim
sudo mkdir -p /var/www/your_domain/html
sudo chown -R www-data:www-data /var/www/your_domain/html
sudo chmod -R 755 /var/www/your_domain
```
### Create test file
```vim
sudo nano /var/www/your_domain/html/index.html
<html>
    <head>
        <title>Welcome to your_domain!</title>
    </head>
    <body>
        <h1>Success!  The your_domain server block is working!</h1>
    </body>
</html>
```
### Create site
```nginx
sudo nano /etc/nginx/sites-available/your_domain

/etc/nginx/sites-available/your_domain

server {
        listen 80;
        listen [::]:80;

        root /var/www/your_domain/html;
        index index.html index.htm index.nginx-debian.html;

        server_name your_domain www.your_domain;

        location / {
                try_files $uri $uri/ =404;
        }
}

sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```
### Redirect Main Domain to Subdomain Https
```nginx
server {
        listen 80;
        server_name iwebitechnology.xyz www.iwebitechnology.xyz *.iwebitechnology.xyz;
        return 301 https://portfolio.iwebitechnology.xyz$request_uri;
        #return 301 https://$host$request_uri;
        #location / {
        #       try_files $uri $uri/ =404;
        #}
}
server {
        listen 443;
        server_name iwebitechnology.xyz www.iwebitechnology.xyz *.iwebitechnology.xyz;
        return 301 https://portfolio.iwebitechnology.xyz$request_uri;
}
```
### Drupal
```nginx
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
}
server {
	listen 80;
	server_name drupal.iwebitechnology.xyz *.drupal.iwebitechnology.xyz;
	return 301 https://drupal.iwebitechnology.xyz$request_uri;

}
server {
	listen 443 ssl;

        server_name drupal.iwebitechnology.xyz *.drupal.iwebitechnology.xyz;

        root /var/www/html/drupal_10_ecommerce;
        index index.php index.html;

	access_log /var/log/nginx/drupal_access.log;
    	error_log  /var/log/nginx/drupal_error.log debug;

	ssl_certificate /etc/cloudflare_ssl/cert.pem;
        ssl_certificate_key /etc/cloudflare_ssl/key.pem;

	client_max_body_size 10M;
 
	location / {
		try_files $uri /index.php?$query_string; # >= Drupal 7
	}
	location @rewrite {
        	rewrite ^/(.*)$ /index.php?q=$1;
    	}
	#block access to hidden files
	location ~ (^|/)\. {
        	return 403;
    	}
	# Don't allow direct access to PHP files in the vendor directory.
    	location ~ /vendor/.*\.php$ {
        	deny all;
        	return 404;
    	}
	location ~ '\.php$|^\/update.php' {
                #ensure php files exists
		fastcgi_split_path_info ^(.+?\.php)(|/.*)$;
		#NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
                include fastcgi_params;
                #block httpoxy attack
		fastcgi_param HTTP_PROXY "";
        	fastcgi_param PATH_INFO $fastcgi_path_info;
        	fastcgi_param QUERY_STRING $query_string;
		#fastcgi_intercept_errors on;
		
		fastcgi_pass unix:/run/php/php8.1-fpm.sock;
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

	location = /favicon.ico {
        	log_not_found off;
        	access_log off;
    	}
	location = /robots.txt {
        	allow all;
        	log_not_found off;
        	access_log off;
    	}
	location ~ \..*/.*\.php$ {
        	return 403;
    	}

   	location ~ ^/sites/.*/private/ {
        	return 403;
    	}
    	# Block access to scripts in site files directory
    	location ~ ^/sites/[^/]+/files/.*\.php$ {
        	deny all;
    	}
	#fighting with styles ?
	location ~ ^/sites/.*/files/styles/ { # For Drupal >= 7
        	try_files $uri @rewrite;
    	}
	#handle private files through drupal
	location ~ ^(/[a-z\-]+)?/system/files/ { # For Drupal >= 7
        	try_files $uri /index.php?$query_string;
    	}
	location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        	try_files $uri @rewrite;
        	expires max;
        	log_not_found off;
    	}
	#enforce clean url
	if ($request_uri ~* "^(.*/)index\.php(.*)") {
        	return 307 $1$2;
    	}
	# Reference: https://alexanderallen.medium.com/series-part-iv-optimizing-nginx-for-drupal-8-x-and-php-7-x-976b541f3768

}
```
### Mediawiki
```nginx
#/etc/nginx/site-available/mediawiki-dev
#ln -s /etc/nginx/site-available/mediawiki-dev /etc/nginx/site-enabled/

server {
        listen 80;
	server_name dev.website.ph *.dev.website.ph;
        return 301 https://dev.website.ph$request_uri; #Redirect to https and retain the URL format
 }
server {
        listen 443 ssl;

        server_name dev.website.ph *.dev.website.ph;
        error_log  /var/log/nginx/mediawiki_error.log;
        root /var/www/html/website;
        index index.php;

        #RSA certificate
        ssl_certificate /etc/letsencrypt/live/dev.website.ph/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/dev.website.ph/privkey.pem;

        include /etc/letsencrypt/options-ssl-nginx.conf;

        #location / {
        #       try_files $uri $uri/ =404;
        #}

        location / {
                try_files $uri $uri/ @rewrite;
        }

        location @rewrite {
                rewrite ^/(.*)$ /index.php?title=$1&$args;
        }

        location ^~ /maintenance/ {
                return 403;
        }

        location /rest.php {
                try_files $uri $uri/ /rest.php?$args;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
                #https://gist.github.com/ikennaokpala/5792a71cfae6818035eedc8abd9ae7b4
                fastcgi_buffers 4 16k; #8|16|32
                fastcgi_buffer_size 16k; #8|16|32
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
                try_files $uri /index.php;
                expires max;
                log_not_found off;
        }

        location = /_.gif {
                expires max;
                empty_gif;
        }

        location ^~ /cache/ {
                deny all;
        }

        # location /dumps {
        # root /var/www/mediawiki/local;
        # autoindex on;
        # }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #       deny all;
        #}
}
```
### WORDPRESS SSL (FOLDER)
default
```nginx
server {
	listen 80;
	#listen [::]:80 default_server;
	server_name iwebitechnology.xyz www.iwebitechnology.xyz *.iwebitechnology.xyz;
	#server_name _;
	return 301 https://portfolio.iwebitechnology.xyz$request_uri;
	#return 301 https://$host$request_uri;
	#location / {
	#	try_files $uri $uri/ =404;
	#}
}
server {
	listen 443;
	server_name iwebitechnology.xyz www.iwebitechnology.xyz *.iwebitechnology.xyz;
	return 301 https://portfolio.iwebitechnology.xyz$request_uri;
}
```
wp_portfolio
```nginx
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
    gzip_types       text/plain application/x-javascript text/xml text/css application/xml;
}
server {
        listen 80;
        server_name portfolio.iwebitechnology.xyz *.portfolio.iwebitechnology.xyz;
        return 301 https://portfolio.iwebitechnology.xyz$request_uri; #Redirect to https and retain the URL format
}
server {
	listen 443 ssl;

        server_name portfolio.iwebitechnology.xyz *.portfolio.iwebitechnology.xyz;

        root /var/www/html/wp_portfolio;
        index index.php index.html;

	access_log /var/log/nginx/wpportfolio_access.log;
    	error_log  /var/log/nginx/wpportfolio_error.log debug;

    	#RSA certificate
        #ssl_certificate /etc/letsencrypt/live/portfolio.iwebitechnology.xyz/fullchain.pem;
        #ssl_certificate_key /etc/letsencrypt/live/portfolio.iwebitechnology.xyz/privkey.pem;
	#include /etc/letsencrypt/options-ssl-nginx.conf;

	ssl_certificate /etc/cloudflare_ssl/cert.pem;
        ssl_certificate_key /etc/cloudflare_ssl/key.pem;
        
	#Start WP Super Cache
	set $cache_uri $request_uri;

    	# POST requests and URLs with a query string should always go to PHP
    	if ($request_method = POST) {
        	set $cache_uri 'null cache';
    	}  
    	if ($query_string != "") {
        	set $cache_uri 'null cache';
    	}   

    	# Don't cache URIs containing the following segments
    	if ($request_uri ~* "(/wp-admin/|/xmlrpc.php|/wp-(app|cron|login|register|mail).php
                          |wp-.*.php|/feed/|index.php|wp-comments-popup.php
                          |wp-links-opml.php|wp-locations.php |sitemap(_index)?.xml
                          |[a-z0-9_-]+-sitemap([0-9]+)?.xml)") {

        	set $cache_uri 'null cache';
    	}  
	
    	# Don't use the cache for logged-in users or recent commenters
    	if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+
                         |wp-postpass|wordpress_logged_in") {
        	set $cache_uri 'null cache';
    	}

	# Use cached or actual file if it exists, otherwise pass request to WordPress
    	location / {
        	try_files /wp-content/cache/supercache/$http_host/$cache_uri/index.html 
			  #.htaacess support and default wordpress redirection if cache not available
			  $uri $uri/ /index.php?$args;
    	} 
	#End WP Super Cache

        location = /favicon.ico {
                log_not_found off;
                access_log off;
        }

        location = /robots.txt {
                allow all;
                log_not_found off;
                access_log off;
        }

        #location / {
        #        try_files $uri $uri/ /index.php?$args;
        #}

	#location / {
        #	try_files $uri $uri/ /index.php?$args;
	#}
	#Deny Accessing PHP Files
	location ~* (/wp-content/.*\.php$|/wp-includes/.*\.php$|/xmlrpc\.php$|/(?:uploads|files)/.*\.php$|/\.ht|^/\.user\.ini) {
            deny all;
            access_log off;
            log_not_found off;
        }
	#Deny Access XMLRPC
	location = /xmlrpc.php {
    		deny all;
		access_log off;
    		log_not_found off;
    		return 404;
	}
        location ~ \.php$ {
		#fastcgi_split_path_info ^(/wp_hotel_booking)(/.*)$; #subdirectory
                #NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
                include fastcgi_params;
                fastcgi_intercept_errors on;
                #fastcgi_pass php;
		#fastcgi_pass unix:/run/php/php7.4-fpm.sock;
                fastcgi_pass unix:/run/php/php8.3-fpm.sock;
		fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        #location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        #        expires max;
        #        log_not_found off;
        #}
	#Start WP Super Cache
	# Cache static files for as long as possible
    	location ~* \.(ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|css|rss|atom|js|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)$ {
        	expires max;
        	log_not_found off;
        	access_log off;
    	}
	#End WP Super Cache
}
```
### IMPROVE PHP FOR NGINX
Check number of processor for worker_processes <number of processor>

$grep processor /proc/cpuinfo | wc -l

Check number of allowed connections for worker_processes <connection>;

$ulimit -n

worker_processes x worker_connections = simultaneous client connections
```nginx
# WORKERS
use www-data;
#worker_processes auto; #default
worker_processes 1;
event {
    worker_connections 1024;
}

# LONGER CACHE EXPIRATION
location ~* .(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 365d;
}

# BUFFER SETTING FOR PHP-FPM
location ~ \.php$ {
    fastcgi_pass unix:/run/php/php7.4-fpm.sock;
    #https://gist.github.com/ikennaokpala/5792a71cfae6818035eedc8abd9ae7b4
    fastcgi_buffers 4 16k; #8|16|32
    fastcgi_buffer_size 16k; #8|16|32
}
        
# GZIP COMPRESSION
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
    gzip_types       text/plain application/x-javascript text/xml text/css application/xml;
}

# OTHER
server {
    client_max_body_size 100m; # INCREASE FILE UPLOAD
    client_body_timeout 60; # INCREASE FILE UPLOAD TIMEOUT 
                            # php.ini
                            # upload_max_filesize = 100M
                            # post_max_size = 100M
    keepalive_timeout 70; # LONGER CONNECTION OF CLIENT
}
```
### IMPROVE PHP-FPM
[PHP-FPM Process Calculator](https://spot13.com/pmcalculator/)
    
[Analyse and Calculate php-fpm runner](https://chrismoore.ca/2018/10/finding-the-correct-pm-max-children-settings-for-php-fpm/)
```vim
#ondemand - auto scaling of 
#dynamic - configured scaling 
    
[Total Available RAM] - [Reserved RAM] - [10% buffer] = [Available RAM for PHP]

Results:
[Available RAM for PHP] / [Average Process Size] = [max_children]
pm.max_children = [max_children]
pm.start_servers = [25% of max_children]
pm.min_spare_servers = [25% of max_children]
pm.max_spare_servers = [75% of max_children]   


```
### SELF SIGNED NGINX
```vim
//OPTIONAL (FOR APACHE)
$ sudo a2enmod rewrite
$ sudo a2enmod ssl
$ sudo a2enmod headers
//GENERATE SELF SIGNED
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

//Nginx config
#RSA certificate
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key
```
### Troubleshooting
502 Bad Gateway
```vim
You may need to increase the buffering
fastcgi_buffers 4 16k;
fastcgi_buffer_size 16k;
```
[Pages Response Slow 502 Response](https://stackoverflow.com/questions/52057345/dns-and-nginx-server-setup-causes-slow-server-and-502-response)
```vim
#Dahil sa 404 page na generated nang PHP program. 
#Nag reredirect ang Page nang Paulit-ulit.

try_files $uri $uri/ /index.php?$args; 

#Solution:
1. Gumawa nang static page(404.html) para sa 404 Error. Mas mabilis mag serve.
    error_page 404 /404.html; 
2. Check Newtork Issue.
3. Issue on DNS Resolution
```
Redirect to https and retain the URL format
```nginx
return 301 https://dev.mediawiki.ph$request_uri;
```
### INSTALL PHP
```vim
sudo apt install php-fpm
NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
```
### NginX Tutorial Links

[9-tips-for-improving-wordpress-performance-with-nginx](https://www.nginx.com/blog/9-tips-for-improving-wordpress-performance-with-nginx)

[Nginx Content Cache](https://docs.nginx.com/nginx/admin-guide/content-cache/content-caching/)

### Nginx Sample

[OSTICKET](https://www.nginx.com/resources/wiki/start/topics/recipes/osticket/)

[LIST OF SAMPLES](https://www.nginx.com/resources/wiki/start/#full-stack-howtos)

### Setup PHP FPM Pool Size

[Configure php-fpm](https://www.digitalocean.com/community/tutorials/php-fpm-nginx#3-configure-nginx-for-php-fpm)

[Determine number of child processes](https://www.kinamo.be/en/support/faq/determining-the-correct-number-of-child-processes-for-php-fpm-on-nginx)

[Compute number of child processes](https://gist.github.com/holmberd/44fa5c2555139a1a46e01434d3aaa512)

### Nginx M1

[nginx sites-available folder not found](https://gist.github.com/sanrandry/bd4350a591f62eb259e48cd9fbfcd642)

