Installing apache
=================

To install web server type
```
yum install -y logrotate httpd httpd-tools mod_ssl
```

Then enable http and https traffic.
```
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

