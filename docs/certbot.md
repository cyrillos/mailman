Installing certbot
==================

To provide https access to the server and support
tls in post service we need to setup certificates.
```
yum install certbot-nginx
```

Initiate procedure
```
certbot --nginx -d dev.tarantool.org
certbot --nginx -d lists.tarantool.org
```

and follow the instructions. In result the
certificates for the domain will be in `/etc/letsencrypt`
directory.

FIXME
-----
Implement autoupdate.
