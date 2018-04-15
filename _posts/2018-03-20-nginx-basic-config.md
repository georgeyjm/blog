---
layout:      post
title:       'Nginx Server Basic Configuration'
subtitle:    'Server Blocks, TLS Requests and uWSGI Apps'
date:        2017-08-21 22:36:47
author:      'George Yu'
header-img:  'img/posts/nginx-basic-config/cover.jpg'
tags:
    - Server
---

OS：CentOS 7.4

## Setup

`su -c 'yum update'` or `sudo yum -y update`

## Nginx

### Install

```shell
yum install epel-release
yum install nginx
systemctl start nginx
systemctl enable nginx
```

### Setup Server Blocks

Add a user for the nginx server.

```shell
useradd nginxuser
passwd nginxuser # then set your password
```

Setup

```shell
mkdir /etc/nginx/sites-available
mkdir /etc/nginx/sites-enabled
vi /etc/nginx/nginx.conf
```

Enter the following lines to the end of the `http {}` block

```conf
include /etc/nginx/sites-enabled/*.conf;
server_names_hash_bucket_size 64;
```

### Setup an Server Block

Let's say we are creating a server block for the URL `georgeyu.cn`. You may use the same steps just changing the name of the URL to add other server blocks.

```shell
mkdir -p /var/www/georgeyu.cn/public_html
vi /var/www/georgeyu.cn/public_html/index.html # or upload your files
chown -R nginxuser:nginxuser /var/www/georgeyu.cn/public_html
chmod 755 /var/www/georgeyu.cn/public_html
vi /etc/nginx/sites-available/georgeyu.cn.conf
```

Enter the following lines into the config file.

```conf
server {
  listen       80;
  server_name  georgeyu.cn www.georgeyu.cn;
  
  location / {
    root   /var/www/georgeyu.cn/public_html;
    index  index.html index.htm;
  }
  
  error_page 404 /errors/404.html;
    location = /errors/40x.html {
      root /var/www/georgeyu.cn/public_html;
  }
  error_page 500 502 503 504 /errors/50x.html;
    location = /errors/50x.html {
      root /var/www/georgeyu.cn/public_html;
  }
}
```

Create a symbolic link of the config file.

```shell
ln -s /etc/nginx/sites-available/georgeyu.cn.conf /etc/nginx/sites-enabled/georgeyu.cn.conf
```

Restart nginx service.

```shell
systemctl restart nginx
```

Create error pages.

```shell
mkdir /var/www/georgeyu.cn/public_html/errors
vi /var/www/georgeyu.cn/public_html/errors/404.html
vi /var/www/georgeyu.cn/public_html/errors/50x.html
```

### TLS Server and HTTPS Requests

Create and upload certificate files (`.key` and `.pem`).

```shell
mkdir /etc/nginx/cert
```

Create an HTTPS server block.

```shell
vi /etc/nginx/sites-available/georgeyu.cn.conf
```

Add the following lines to the end of the file.

```conf
server {
  listen       443 ssl;
  server_name  georgeyu.cn www.georgeyu.cn;
  
  ssl                        on;
  ssl_certificate            /etc/nginx/cert/214331526280345.pem;
  ssl_certificate_key        /etc/nginx/cert/214331526280345.key;
  ssl_session_timeout        5m;
  ssl_ciphers                ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
  ssl_protocols              TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers  on;
  
  location / {
    root   /var/www/georgeyu.cn/public_html;
    index  index.html index.htm;
  }
  
  error_page 404 /errors/404.html;
    location = /errors/40x.html {
      root /var/www/georgeyu.cn/public_html;
  }
  error_page 500 502 503 504 /errors/50x.html;
    location = /errors/50x.html {
      root /var/www/georgeyu.cn/public_html;
  }
}
```

Add `301 Moved Permanently` redirect to `HTTP` requests.

```shell
vi /etc/nginx/sites-available/georgeyu.cn.conf
```

Add the following line after the `server_name` line of the `listen 80` (HTTP) server block, then remove anything below it in this server block.

```conf
return 301 https://$server_name$request_uri;
```

Restart Nginx.

```shell
systemctl restart nginx
```

## Python & Application Deployment

Following [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-centos-7).

### Install Python and Other Dependencies

In this example, we are creating an app called `weeservices` and it can be accessed by `weestudios.org/services`.

```shell
yum install yum-utils
yum groupinstall development
yum install https://centos7.iuscommunity.org/ius-release.rpm
yum install python36u-pip python36u-devel gcc
```

### Setup Virtual Environment

```shell
mkdir -p /var/www/weestudios.org/apps/weeservices
cd /var/www/weestudios.org/apps/weeservices
python3.6 -m venv weeservicesenv
. weeservicesenv/bin/activate
```

### Install Packages

```shell
pip install flask uwsgi
```

### Setup Flask Application

```shell
vi weeservices.py
```

Copy the following content into the file.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return '<h1 style="color: blue">Hello There!</h1>'
    
@app.route('/services')
def services():
    return '<h1 style="color: blue">Hello There! I am service</h1>'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

### Create WSGI Entry Point

```shell
vi wsgi.py
```

Copy the following content into the file.

```python
from weeservices import app

if __name__ == '__main__':
    app.run()
```

Test the WSGI entry point.

```shell
uwsgi --socket 0.0.0.0:8000 --protocol=http --callable app -w wsgi
```

Exit the environment.

```shell
deactivate
```

Create uWSGI configuration file.

```shell
vi weeservices.ini
```

Copy the following content into the file.

```ini
[uwsgi]
module = wsgi
callable = app

master = true
processes = 5

socket = weeservices.sock
chmod-socket = 660
vacuum = true

die-on-term = true
```

Create Systemd unit file.

```shell
vi /etc/systemd/system/weeservices.service
```

Copy the following content into the file.

```ini
[Unit]
Description=uWSGI instance to serve weeservices
After=network.target

[Service]
User=nginxuser
Group=nginx
WorkingDirectory=/var/www/weestudios.org/apps/weeservices
Environment="PATH=/var/www/weestudios.org/apps/weeservices/weeservicesenv/bin"
ExecStart=/var/www/weestudios.org/apps/weeservices/weeservicesenv/bin/uwsgi --ini weeservices.ini

[Install]
WantedBy=multi-user.target
```

Change owner and permission.

```shell
chown -R nginxuser:nginxuser /var/www/weestudios.org/apps
chmod 755 /var/www/weestudios.org/apps
```

Start the service.

```shell
systemctl start weeservices
systemctl enable weeservices
```

Configure Nginx to proxy requests.

```shell
vi /etc/nginx/sites-available/weestudios.org.conf
```

Add the following lines before the existing `location` block in the `server` block.

```conf
  location /services {
    include     uwsgi_params;
    uwsgi_pass  unix:/var/www/weestudios.org/apps/weeservices/weeservices.sock;
  }
```

Restart Nginx.

```shell
systemctl restart nginx
```