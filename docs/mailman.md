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

For some reason on Fedora 33 there is no spawn-fcgi serive by default,
so run SysV script first.
```
/etc/init.d/spawn-fcgi start
```

Which will make the service generated. Run and enable it.
```
systemctl start spawn-fcgi
systemctl enable spawn-fcgi
chkconfig --add spawn-fcgi
```

In `/etc/mailman/mm_cfg.py` define
```
DEFAULT_URL_HOST    = 'lists.tarantool.org'
DEFAULT_EMAIL_HOST  = 'dev.tarantool.org'
DEFAULT_URL_PATTERN = 'https://%s/mailman/'
PUBLIC_ARCHIVE_URL  = 'https://%(hostname)s/pipermail/%(listname)s'
MTA                 = 'Postfix'
REMOVE_DKIM_HEADERS = Yes
```

The case why `DEFAULT_URL_HOST` is different from
`DEFAULT_EMAIL_HOST` is that initially we wanted to
run mailman under `https://dev.tarantool.org/mailman`
but it turned out that such urls are not really friendly
for the mailman so that lists are placed under
`https://lists.tarantool.org/`.

And enable the mailman
```
systemctl enable mailman
```

In `/etc/nginx/fastcgi.conf` comment out
```
#fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
```

In `/etc/nginx/nginx.conf`, add the following `server` section
```
    server {
        listen          443 ssl;
        server_name     lists.tarantool.org;

        location = / {
            rewrite ^ /mailman/listinfo permanent;
        }

        location = /mailman {
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

        ssl_certificate /etc/letsencrypt/live/lists.tarantool.org/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/lists.tarantool.org/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    }
```

And change the user nginx starts as
```
user nginx apache;
```

Make sure the `nginx` user is in `mailman` group.
```
sermod -aG mailman nginx
```
Otherwise `nginx` will not be able to read achives.

Since we're accessing mailman via `lists.tarantool.org/mailman` subpath
define a symlink
```
ln -s /usr/lib/mailman/cgi-bin /usr/lib/mailman/cgi-bin/mailman
chown root:mailman /usr/lib/mailman/cgi-bin/mailman
```

Create site global password (which is similar to root user in system)
```
/usr/lib/mailman/bin/mmsitepass
New site password:
Again to confirm password:
Password changed.
```

Create the list creator password
```
/usr/lib/mailman/bin/mmsitepass -c
```

See [manual](https://www.gnu.org/software/mailman/mailman-install/node44.html)
for more details

Create a new site global list (if not created during installation)
```
/usr/lib/mailman/bin/newlist mailman
Enter the email of the person running the list: gorcunov@tarantool.org
Initial mailman password:
Hit enter to notify mailman owner...
```

Due to syslinux accessing `https://lists.tarantool.org/mailman`
will cause a problem thus modify the rules
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
systemctl enable nginx
```

Once the web part is operating we need to adjust `postfix` to process
mailing lists mails.

Run
```
/usr/lib/mailman/bin/genaliases
postalias /etc/mailman/aliases
```

which will generate `/etc/mailman/aliases.db`.

Now we need to hook this aliases to `postfix` via `main.cfg`.
```
alias_maps = hash:/etc/aliases,hash:/etc/mailman/aliases
```

And restart `postfix`
```
systemctl restart postfix
```

Note that every time new list is created via mailman interface
we need to update `/etc/mailman/aliases` map and restart postfix.
