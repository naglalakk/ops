Wordpress + Nginx + FastCGI
===

Update system packages

    sudo apt update

### Mysql

Install mysql and setup user

    sudo apt install mysql-server
    sudo mysql_secure_installation

Once you've created and validated the password do

    sudo mysql

Run

    CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';

where user is your desired username and password is your unique, validated password

GRANT ALL while you're at it

    GRANT ALL PRIVILEGES ON *.* TO user@localhost;

Create a database for your app 

    CREATE DATABASE app;

### Wordpress

Install your application on /var/www/

    sudo mkdir /var/www/app && sudo chown -R www-data:www-data /var/www/app

Get the latest Wordpress version

    curl https://wordpress.org/latest.tar.gz -o latest.tar.gz

Extract the content and move to /var/www/

    tar -xvsf latest.tar.gz && \
      sudo mv wordpress/* /var/www/app/ && \
      sudo chown -R www-data:www-data /var/www/app


Tamper with the wp-config. You can use [The Salt
Generator](https://api.wordpress.org/secret-key/1.1/salt/) to generate unique salts.

### PHP

Install php deps

    sudo apt install php php-cli php-fpm php-json php-pdo php-mysql php-zip php-gd  php-mbstring php-curl php-xml php-pear php-bcmath

Disable apache server

    sudo systemctl disable --now apache2


In /etc/php/7.4/fpm/php-fpm.conf add the following lines

    emergency_restart_threshold 10
    emergency_restart_interval 1m
    process_control_timeout 10s


In /etc/php/7.4/fpm/pool.d/www.conf add this

    listen.backlog = 65536 # default is 511;

    pm = dynamic
    pm.max_children = 75
    pm.start_servers = 25
    pm.min_spare_servers = 15
    pm.max_spare_servers = 50
    pm.max_requests = 200


Remember to 

    sudo service php7.4-fpm restart

after changes ;) 

### Nginx

First, increase max open files.
In /etc/sysctl.conf, add the following

    fs.file-max=65536

Now in /etc/security/limits.conf add the following

    * soft nofile 65536
    * hard nofile 65536

Install nginx

    sudo apt install nginx

Allow connections on port 80, 443

    sudo ufw allow proto tcp from any to any port 80,443


Install this package for fastcgi_purge_cache function in nginx

    sudo apt install libnginx-mod-http-cache-purge

Update nginx.conf, located at /etc/nginx/nginx.conf

    # /etc/nginx/nginx.conf
    worker_rlimit_nofile 65536;

    events {
        worker_connections 8192;
        multi_accept on; # uncomment this
    }

    http {
        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 2 2;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        client_max_body_size 2000M;
        client_body_buffer_size 1m; # was 10K
        client_header_buffer_size 1k;
        large_client_header_buffers 4 16k;
        types_hash_max_size 2048;
        
        ##
        # File Cache Settings
        ##

        open_file_cache max=2000 inactive=20s;
        open_file_cache_valid 60s;
        open_file_cache_min_uses 5;
        open_file_cache_errors off;

        ##
        # SSL Settings
        ##
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Gzip Settings
        ##

        gzip on;
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_http_version 1.1;
        gzip_buffers 16 8k;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    }


Next in your app nginx config

    # Note, /etc/nginx/cache has to be created with www-data permission
    fastcgi_cache_path /etc/nginx/cache levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";


    server {
        server_name yourapp.example.com;
        root /var/www/app;
        index index.php

        set $skip_cache 0;

        access_log /var/log/nginx/app.access.log;
        error_log /var/log/nginx/app.error.log;

        # POST requests and URLs with a query string should always go to PHP
        if ($request_method = POST) {
            set $skip_cache 1;
        }

        if ($query_string != "") {
            set $skip_cache 1;
        }

        # Don't cache URIs containing the following segments
        if ($request_uri ~* "/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
            set $skip_cache 1;
        }

        # Don't use the cache for logged-in users or recent commenters
        if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass
            |wordpress_no_cache|wordpress_logged_in") {
            set $skip_cache 1;
        }

        location / {
            try_files $uri $uri/ /index.php?$args;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ \.php$ {
            try_files $uri /index.php =404;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_intercept_errors on;
            fastcgi_cache WORDPRESS;
            fastcgi_cache_valid 200 60m;
            fastcgi_pass unix:/run/php/php7.4-fpm.sock;
            fastcgi_cache_bypass $skip_cache;
            fastcgi_no_cache $skip_cache;
            fastcgi_ignore_client_abort off;
            fastcgi_connect_timeout     3s; # was 60
            fastcgi_buffer_size         128k;
            fastcgi_buffers             128 16k; # was 4 256k
            fastcgi_busy_buffers_size   256k;
            fastcgi_temp_file_write_size 256k;
            reset_timedout_connection on;
            add_header X-FastCGI-Cache $upstream_cache_status;
        }

        # Side note:
        # Install the Nginx Helper Plugin 
        # to purge this cache from the admin dashboard
        location ~ /purge(/.*) {
          fastcgi_cache_purge WORDPRESS "$scheme$request_method$host$1";
        }

        location ~* ^.+.(ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|css|rss|atom|js|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)$ {
            access_log off;
            log_not_found off;
            expires max;
        }

        location = /robots.txt {
            access_log off;
            log_not_found off;
        }

    }

And finally reload nginx
    
    service nginx reload
