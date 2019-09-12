Installing OpenDKIM
===================

Install the package first
```
yum install opendkim
```

Generate keys for `dev.tarantool.org` domain
```
opendkim-genkey -t -D /etc/opendkim/keys/ -s dev -d dev.tarantool.org
```

which will create `/etc/opendkim/keys/dev.txt` and
`/etc/opendkim/keys/dev.private` files.

Make sure the owner and the group is `opendkim` for these files.

The content of `/etc/opendkim/keys/dev.txt` should be added to DNS.
```
dev._domainkey  IN      TXT     ( "v=DKIM1; k=rsa; "
	"p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDPN615"
	"q/Q/lfw0qGp/m40h9sDYEvHhQyQA0ttBKFL8i5mpn5QEIp"
	"rj4VACjgLaP3A3z1MnMr6gkeoil7buqYrfDGxrGu8isl5O"
	"X9NlAuo07LsBfq3fL7naYIrlN4rQ0DvXEJnRuYjWIDJ4Y1"
	"rtuT2bgVXoH9M3i/ZVdA0Smdf/tQIDAQAB" )
```

In `/etc/opendkim/KeyTable` add
```
dev._domainkey.tarantool.org tarantool.org:dev:/etc/opendkim/keys/dev.private
```

In `/etc/opendkim/SigningTable` add
```
*@dev.tarantool.org dev._domainkey.tarantool.org
```

In `/etc/opendkim/TrustedHosts` add
```
127.0.0.1
::1
dev.tarantool.org
```

The `/etc/opendkim.conf` should be
```
PidFile         /var/run/opendkim/opendkim.pid
Mode            sv
Syslog          yes
LogResults      yes
SyslogSuccess   yes
LogWhy          yes
UserID          opendkim:opendkim
Socket          inet:8891@localhost
Umask           002
SendReports     no
SoftwareHeader  yes
Canonicalization        relaxed/simple
Selector        dev
MinimumKeyBits  1024
KeyFile         /etc/opendkim/keys/default.private
KeyTable        refile:/etc/opendkim/KeyTable
SigningTable    refile:/etc/opendkim/SigningTable
InternalHosts   refile:/etc/opendkim/TrustedHosts
OversignHeaders From
```

Enable and start the daemon.
```
systemctl enable opendkim
systemctl restart opendkim
```

Probably should add `spf` and `_dmarc` records in DNS.
```
v=spf1 a mx ip4:95.163.249.249 ~all
_dmarc.dev.tarantool.org v=DMARC1; p=none
```

Finally update `postfix` settings to start signing outgoing mails.
In `/etc/postfix/main.cf` add
```
smtpd_milters           = inet:127.0.0.1:8891
non_smtpd_milters       = $smtpd_milters
milter_default_action   = accept
milter_protocol         = 2
```
