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
