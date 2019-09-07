Installing Postfix
==================

Prepare DNS
-----------
Configure DNS to be able to receive the mail for
subdomain `@dev.tarantool.org`
```
dev IN MX 100 dev.tarantool.org.
```

Make sure that you already have properly configured `A`
and `PTR` records. See [configure os environment](os.md)
section. Note that we will need to adjust DNS records
more for SPF and OpenDKIM sake later.

Test the settings
```
dig -t mx dev.tarantool.org
dev.tarantool.org.	3600	IN	MX	100 dev.tarantool.org.
```

Allow to listen for smtp traffic.
```
firewall-cmd --add-service=smtp --permanent
firewall-cmd --reload
```

Configure Postfix
-----------------
Install the package if it were not yet
```
yum install postfix
```

It is assumed that config files are laying in `/etc/postfix` directory.
