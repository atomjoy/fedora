# Fedora 39 Desktop

Installing and configuring Fedora 39 Workstation with Windows 10.

## Install Workstation Live

You need to add Efi Partition with mount point /boot/efi or the system will not install (disk errors)

```sh
Efi (required partition) at least 1GB mount point: /boot/efi
Root (required partition) at least 20GB mount point: / 
Swap (optional) at least 2GB (2 x RAM) /swap
```

## Boot grub

Boot iso with Windows

### Set grub auto save

```sh
sudo nano /etc/default/grub

# Add
GRUB_SAVEDEFAULT=true
```

### Refresh grub repos

```sh
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

## LEMP

### Install Nginx, Php

```sh
sudo dnf install nginx
sudo dnf install php-fpm php-cli
sudo dnf install mariadb-server
sudo systemctl enable mariadb

# Secure mysql server or set firewall ban on port 3306
# sudo mysql_secure_installation

# Login to mysql with pass
sudo mysql -u root -p
```

### Add user and group for the application

Create user and group without login

```sh
# User, group, no-home, no-login
sudo useradd -r -s /bin/false <appname>_app

# Or long version
# sudo groupadd <appname>_app
# sudo useradd -g <appname>_app <appname>_app
# sudo chsh -s /bin/nologin <appname>_app
```

### Backup old PHP-FPM pool config or copy to new app config

```sh
sudo mv /etc/php-fpm.d/www.conf /etc/php-fpm.d/www.conf.back
```

### Create a custom pool config

An appname.conf file unique for each application

```sh
sudo nano /etc/php-fpm.d/<appname>.conf
```

### Edit config file

Create first linux user and group <appname>_app if not exists

```sh
[<appname>_pool]
; General settings
user = <appname>_app
group = <appname>_app
listen = /var/run/php-fpm/<appname>_pool.sock
# listen = 127.0.0.1:9000
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
; FPM process manager configuration
pm = dynamic
pm.max_children = 50
pm.start_servers = 3
pm.min_spare_servers = 3
pm.max_spare_servers = 10
; Php memory limit, upload
php_admin_value[memory_limit] = 100M
php_admin_value[post_max_size] = 50M
php_admin_value[upload_max_filesize] = 10M
; FPM log config
slowlog = /var/log/php-fpm/<appname>_pool-slow.log
request_slowlog_timeout = 10s
php_admin_value[error_log] = /var/log/php-fpm/<appname>_pool-error.log
php_admin_flag[log_errors] = on
; FPM php config php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
php_value[soap.wsdl_cache_dir] = /var/lib/php/wsdlcache
; Show php errors set to off in production
php_flag[display_errors] = on
; FPM php config goes below
php_value session.cookie_lifetime=0
php_value session.use_cookies=1
php_value session.use_only_cookies=1
php_value session.use_strict_mode=1
php_value session.cookie_httponly=1
; Allow with http set 0
php_value session.cookie_secure=1
php_value session.use_trans_sid=0
; Or allow more with "Lax"
php_value session.cookie_samesite="Strict"
; Allow caching only when the content is not private.
; php_value session.cache_limiter="private_no_expire"
; php_value session.hash_function="sha256"
; Limit session time
php_value session.gc_maxlifetime="3660"
```

### Nginx server conf

```sh
location ~ \.php$ {
  include snippets/fastcgi-php.conf;  
  fastcgi_pass unix:/var/run/php-fpm/<appname>_pool.sock
  # fastcgi_pass unix:/var/run/php-fpm/<appname>_pool.php8.1-fpm.sock
  # fastcgi_pass unix:/run/php-fpm/php8.2-fpm.sock;
  # fastcgi_pass unix:/run/php-fpm/php8.1-fpm.sock;
  # fastcgi_pass 127.0.0.1:9000;
}

upstream php-fpm {
  server unix:/var/run/php-fpm/<appname>_pool.sock
  # server unix:/var/run/php-fpm/<appname>_pool.php8.1-fpm.sock
  # server unix:/run/php-fpm/php8.1-fpm.sock;
  # server unix:/run/php-fpm/php8.2-fpm.sock;
}
```

### Show logs

```sh
tail -f /var/log/php-fpm/*.log
```

### Create app dir

```sh
# Add app dir
sudo mkdir /app/web/<appname>_app

# Set group and permissions
sudo chmod -hR 2755 /app/web/<appname>_app
sudo chown -hR nginx:<appname>_app /app/web/<appname>_app

# Add user to app group
sudo usermod -aG <appname>_app <username>

# Show
ls -ld /app/web/<appname>_app

# At this point, all members of the <appname>_app group can create and edit files in the /app/web/<appname>_app/
# directory without the administrator having to change file permissions every time users write new files.
```

### Create app virtualhost file

```sh
nano /etc/nginx/conf.d/<appname>_app.conf
```

### Edit virtualhost file

```sh
server {
    disable_symlinks off;
    client_max_body_size 100M;
    source_charset utf-8;
    charset utf-8;

    listen 80;
    listen [::]:80;

    server_name <appname_app.example.com>;
    root /app/web/<appname>_app;
    index index.php index.html;

    location / {
      # try_files $uri $uri/ =404;
      try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {        
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php-fpm/<appname>_pool.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Short
    # location ~ \.php$ {
    #  include snippets/fastcgi-php.conf;
    #  fastcgi_pass unix:/var/run/php-fpm/<appname>_pool.sock;
    #  # fastcgi_pass 127.0.0.1:9000;
    # }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        # expires -1;
        expires max;
        log_not_found off;
    }

    access_log /var/log/nginx/<appname>_app.access.log;
    error_log /var/log/nginx/<appname>_app.error.log;    
}
```

### Test config and restart nginx

```sh
sudo nginx -t
sudo systemctl restart nginx
```

## User groups

```sh
# Show
groups <username>

# Create user
sudo useradd -m dummy

# Set password
sudo passwd dummy

# Add user to group
sudo usermod -aG <group> <username>
sudo usermod -aG <group>,<group1>,<group2> <username>

# Remove user from group
sudo gpasswd -d <username> <group>

# Remove user
sudo userdel -r <username>
```

## Firewall desktop

You can remove **firewall-cmd** and install **ufw** or use **iptables-services**

```sh
# Disable and remove
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo dnf remove firewalld

# Run
sudo dnf install firewalld
sudo systemctl status firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --state

# Install GUI
sudo dnf install firewall-config

# List
sudo firewall-cmd --get-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all --zone=drop
sudo firewall-cmd --list-ports --zone=drop

# Set drop for all incoming
sudo firewall-cmd --set-default-zone drop
sudo firewall-cmd --runtime-to-permanent
# Or
sudo firewall-cmd --permanent --set-default-zone drop

# ICMP
sudo firewall-cmd --get-icmptypes
# Is blocked
sudo firewall-cmd --query-icmp-block=<icmptype>
# Block
sudo firewall-cmd --add-icmp-block=<icmptype>
sudo firewall-cmd --add-icmp-block=echo-reply
# Remove
sudo firewall-cmd --remove-icmp-block=<icmptype>
sudo firewall-cmd --remove-icmp-block=echo-reply
# Block all (nie działa dla echo-reply chyba że no to yes)
sudo firewall-cmd --add-icmp-block-inversion
sudo firewall-cmd --runtime-to-permanent

# Open port mysql
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --permanent --remove-port=3306/tcp
sudo firewall-cmd --runtime-to-permanent

# Or with
# sudo iptables -A INPUT -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
```

### Firewall remove rules

```sh
# Remove all
sudo rm -rf /etc/firewalld/zones
sudo rm -rf /etc/firewalld/direct.xml
sudo iptables -X
sudo iptables -F
sudo iptables -Z
sudo systemctl restart firewalld

# Remove zone
sudo firewall-cmd --zone=CUSTOM --remove-service=CUSTOM
```
