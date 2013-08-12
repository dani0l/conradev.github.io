---
layout: post
title: Getting a Flask website up and running in Ubuntu
tags: 
---

This is a guide to get a Flask website up and running on Ubuntu 12.04 LTS using nginx and uWSGI. There are [many routes](http://me.veekun.com/blog/2012/05/05/python-faq-webdev/) to take when it comes to Python on the web; this just is my personal favorite. Some people enjoy configuring servers, while others view it as a chore. Regardless, this guide should get you up, running, and ready to make something awesome in no time!

# Installation

## nginx

To install nginx you first need to add the repository. Add the following to `/etc/apt/sources.list.d/nginx-lucid.list` :

```
deb http://nginx.org/packages/ubuntu/ lucid nginx
deb-src http://nginx.org/packages/ubuntu/ lucid nginx
```

You will also want to add the gpg key to the `apt` keyring:

``` bash
wget http://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key
rm nginx_signing.key
```

Finally, to install nginx, run:

``` bash
apt-get update
apt-get install nginx
```

## uWSGI

You can use pip to install the latest version of uWSGI by doing the following:

``` bash
sudo apt-get install python-dev build-essential python-pip
sudo pip install uwsgi
```

To configure uWSGI to run as a daemon, you first want to create a separate `uwsgi` user:

``` bash
sudo useradd -c 'uwsgi user,,,' -g nginx -d /nonexistent -s /bin/false uwsgi
```

You then want to create an upstart configuration file, to run uWSGI in the background. Add the following to `/etc/init/uwsgi.conf` :

```
description "uWSGI"
start on runlevel [2345]
stop on runlevel [06]

respawn

exec uwsgi --master --processes 4 --die-on-term --uid uwsgi --gid nginx --socket /tmp/uwsgi.sock --chmod-socket 660 --no-site --vhost --logto /var/log/uwsgi.log
```

To set up logging, you can add a logrotate configuration file at `/etc/logrotate.d/uwsgi`:

```
/var/log/uwsgi.log {
    rotate 10
    daily
    compress
    missingok
    create 640 uwsgi adm
    postrotate
        initctl restart uwsgi >/dev/null 2>&1
    endscript
}
```

and to prime the log file, you can run:

``` bash
sudo touch /var/log/uwsgi.log
sudo logrotate -f /etc/logrotate.d/uwsgi
```

It is okay if there is an error with the postrotate script. uWSGI cannot be restarted because it is not currently running.

## virtualenv

Using virtualenv is a good idea because it compartmentalizes the environment of your application. To install virtualenv, you can use pip:

``` bash
sudo pip install virtualenv
```

# Configuration

## Flask

Now to set up the website! In this case, the website is going to be called 'helloworld' and it is going to be set up in `/srv/www/helloworld`. Feel free to change it, just adjust these instructions accordingly.

The first step is to create the directory:

``` bash
sudo mkdir -p /srv/www/helloworld
```

and a subdirectory for static files, to be hosted by nginx:

``` bash
cd /srv/www/helloworld
mkdir static
```

The next step is to create a virtual environment for the application to run in:

``` bash
virtualenv ./env
```

and to install Flask in that environment:

``` bash
source env/bin/activate
pip install Flask
deactivate
```

Now that that is done, let's create a sample 'Hello World' application, in `application.py`:

``` python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run()
```

The permissions of the directory now have to be configured. uWSGI needs read permission to read the contents of the scripts, and write permission to save compiled python files:

``` bash
sudo usermod -a -G nginx $USER
sudo chown -R $USER:nginx /srv/www/helloworld
sudo chmod -R g+w /srv/www/helloworld
```

## Nginx

The final step is to configure nginx. First, remove the default configuration file:

``` bash
sudo rm /etc/nginx/conf.d/default.conf
```

Now, add the helloworld configuration file at `/etc/nginx/conf.d/helloworld.conf`:

``` nginx
server {
    listen       80;
    server_name  localhost;

    location /static {
        alias /srv/www/helloworld/static;
    }

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/tmp/uwsgi.sock;
        uwsgi_param UWSGI_PYHOME /srv/www/helloworld/env;
        uwsgi_param UWSGI_CHDIR /srv/www/helloworld;
        uwsgi_param UWSGI_MODULE application;
        uwsgi_param UWSGI_CALLABLE app;
    }

    error_page   404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## Running it

Now that you have successfully configured your website, you can run it using:

``` bash
sudo service uwsgi restart
sudo service nginx restart
```

# Extensions

## Flask

Flask is fantastic because it is very extensible. Check out the [documentation](http://flask.pocoo.org/docs/) to get started.

## More Websites!

It is really easy to add more websites to this configuration. Simply create a new application directory, set it up, and create a second nginx configuration. uWSGI is configured to be in virtualhost mode, so it can handle multiple websites at once.

## Portability

You can also use uWSGI as an http server, for testing purposes. If you have a copy of your website checked out in the current directory, you can run

``` bash
uwsgi --http 127.0.0.1:9090 --pyhome ./env --module application --callable app
```

and uWSGI will create a HTTP server hosting your application on port 9090. This is incredibly useful for development.

Discuss on Hacker News [here](http://news.ycombinator.com/item?id=3937691)