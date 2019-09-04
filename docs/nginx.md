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
