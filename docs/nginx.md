Installing nginx
================

To install web server type
```
yum install epel-release
yum install nginx
```

Then enable http and https traffic.
```
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

The pages will be hosted at `/var/www` with `nginx` user and group.
```
mkdir -p /var/www/cgi-bin
chown nginx:nginx /var/www/cgi-bin
chown nginx:nginx /var/www
```

Do not forget to setup SELinux rules.
```
semanage fcontext -a -t httpd_sys_content_t "/var/www(/.*)?"
semanage fcontext -a -t httpd_sys_script_exec_t "/var/www/cgi-bin(/.*)?"
restorecon -r -v /var/www
```

Put the following content to `/etc/nginx.conf` file
```
user nginx nginx;
worker_processes 1;
error_log /var/log/nginx/error.log info;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
    use epoll;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    server_tokens               off;

    client_header_timeout       10m;
    client_body_timeout         10m;
    send_timeout                10m;

    connection_pool_size        256;
    client_header_buffer_size   1k;
    large_client_header_buffers 4 2k;
    request_pool_size           4k;

    gzip                    on;
    gzip_min_length         1100;
    gzip_buffers            4 8k;
    gzip_types              text/plain;

    sendfile                on;
    tcp_nopush              on;
    tcp_nodelay             on;
    types_hash_max_size     2048;

    keepalive_timeout       75 20;
    ignore_invalid_headers  on;

    include                 /etc/nginx/mime.types;
    default_type            application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       *:80;
        server_name  dev.tarantool.org;

        include /etc/nginx/default.d/*.conf;

        index   index.html;
        root    /var/www;

        location / {
        }
    }
}
```

After that put some content into /var/www/index.html which
should appear at http://dev.tarantool.org address.

Next setup [certbot](certbot.md) to process configuring https
access. Once configured continue from this point.

Once certificates are generated the `server` section above
should be changed to the following
```
    server {
        listen          80 default_server;
        server_name     _;
        return 301 https://$host$request_uri;
    }

    server {
        listen          443 ssl;
        server_name     dev.tarantool.org;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        index   index.html;
        root    /var/www;

        location / {
        }

        ssl_certificate /etc/letsencrypt/live/dev.tarantool.org/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/dev.tarantool.org/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    }
```

In result all http traffic gonna be redirected to https instance.
