---
layout: post
title: How to setup an ultra-fast CDN using Nginx and Varnish
date: 2012-03-12
comments: true
tags: cdn nginx varnish
---

If your website takes more than 3 seconds to load, there is a big chance to loose your site visitors and that is a risk
we are dealing with today due to the big competition in the industry.

There are lots of reasons which effects on the site's speed

 - Large multimedia contents
 - DNS look up issue
 - Server response time
 - Multiple redirects

are few from a lengthy list.

If you are using a good CDNs to serve contents, then that will give a good boost while loading the site.
Last month I had to setup up Our own CDN for [WSO2 Oxygentank Developer Portal](http://wso2.org/) and i'm trying to summaries steps that I followed.

###Installing Nginx and varnish

```bash
    apt-get install nginx
    apt-get install varnish
```

###Nginx Config
- /etc/nginx/sites-enabled/default

```
    server {
           open_file_cache_valid 20m;
            listen 81;
            server_name mydomain.com;
            access_log   /var/log/cdn/mydomain.com/logs/access.log;
            root   /cdn_document_path/;

            location ~* \.(gif|jpg|jpeg|png|wmv|avi|mpg|mpeg|mp4|htm|html|js|css|mp3|swf|ico|flv)$ {
                            open_file_cache_valid 120m;
                            expires 7d;
                            open_file_cache max=1000 inactive=20s
            }

             location = /50x.html {
                            root   /var/www/nginx-default;
            }

            # No access to .htaccess files.
            location ~ /\.ht {
                deny  all;
            }
    }
```

###Varnish Config
- /etc/default/varnish

```
    START=yes
    NFILES=131072
    MEMLOCK=82000
    INSTANCE=$(uname -n)
    DAEMON_OPTS="-a :80 \
                 -T localhost:6082 \
                 -f /etc/varnish/default.vcl \
                 -S /etc/varnish/secret \
                 -s file,/var/lib/varnish/$INSTANCE/varnish_storage.bin,1G"
```

- /etc/varnish/default.vcl

```
    backend default {
        .host = "127.0.0.1";
        .port = "81";
    }

    sub vcl_recv {
     if (req.url ~ “\.(js|css|jpg|jpeg|png|gif|gz|tgz|bz2|tbz|mp3|ogg|swf)$”) {
      return (lookup);
     }
    }

    sub vcl_fetch {
     if (req.url ~ “\.(js|css|jpg|jpeg|png|gif|gz|tgz|bz2|tbz|mp3|ogg|swf)$”) {
      unset obj.http.set-cookie;
     }
    }
```
###Start Nginx and Varnish

```bash
    /etc/init.d/nginx start
    /etc/init.d/varnish restart
```

Once you start Nginx and Varnish. Varnish will start to server port 80 web traffic.
If it is a static content varnish will server and Nginx will look after all dynamic contents.