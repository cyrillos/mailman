Installing Mailman2
===================

Install the packages

```
yum install mailman fcgiwrap spawn-fcgi
```

It will install `httpd` as a dependency but we won't use it
but configure `nginx` instead. Same time the group `apache`
will be created which is required by `mailman`.
```
systemctl disable httpd
```

Configure `/etc/sysconfig/spawn-fcgi`
```
FCGI_SOCKET=/var/run/fcgiwrap.socket
FCGI_PROGRAM=/sbin/fcgiwrap
FCGI_USER=nginx
FCGI_GROUP=apache
FCGI_EXTRA_OPTIONS="-M 0700"
OPTIONS="-u $FCGI_USER -g $FCGI_GROUP -s $FCGI_SOCKET -S $FCGI_EXTRA_OPTIONS -F 1 -P /var/run/spawn-fcgi.pid -- $FCGI_PROGRAM"
```

Run and enable it
```
systemctl start spawn-fcgi
chkconfig --add spawn-fcgi
```

In `/etc/mailman/mm_cfg.py` define
```
DEFAULT_URL_HOST    = 'dev.tarantool.org'
DEFAULT_EMAIL_HOST  = 'dev.tarantool.org'
DEFAULT_URL_PATTERN = 'https://%s/mailman/'
PUBLIC_ARCHIVE_URL  = 'https://%(hostname)s/archives/%(listname)s'
MTA                 = 'Postfix'
```

And enable the mailman
```
systemctl enable mailman
```

In `/etc/nginx/fastcgi.conf` comment out
```
#fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
```

In `/etc/nginx/nginx.conf`, add the following to the `server` section
```
        location = /mailman {
            rewrite ^ /mailman/listinfo permanent;
        }

        location = /mailman/ {
            rewrite ^ /mailman/listinfo permanent;
        }

        location /mailman {
            root  /usr/lib/mailman/cgi-bin/;
            fastcgi_pass  unix:/var/run/fcgiwrap.socket;
            fastcgi_intercept_errors on;
            include /etc/nginx/fastcgi_params;

            fastcgi_split_path_info (^/mailman/[^/]+)(/.*)$;
            fastcgi_param SCRIPT_FILENAME   $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO         $fastcgi_path_info;
        }

        location /icons {
            alias /usr/lib/mailman/icons;
        }

        location /pipermail {
            alias /var/lib/mailman/archives/public;
            autoindex on;
        }
```

Since we're accessing mailman via `dev.tarantool.org/mailman` subpath
define a symlink
```
ln -s /usr/lib/mailman/cgi-bin /usr/lib/mailman/cgi-bin/mailman
chown root:mailman /usr/lib/mailman/cgi-bin/mailman
```

Create a new list
```
/usr/lib/mailman/bin/newlist mailman
Enter the email of the person running the list: gorcunov@tarantool.org
Initial mailman password: 
Hit enter to notify mailman owner...
```

Due to syslinux accessing `https://dev.tarantool.org/mailman` will cause
a problem thus modify the rules
```
grep nginx /var/log/audit/audit.log | audit2allow
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
semodule -i nginx.pp
```

Now start mailman
```
systemctl start mailman
systemctl enable mailman
```

Finally reload the nginx
```
systemctl reload nginx
```
