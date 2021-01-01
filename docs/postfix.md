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
more OpenDKIM sake later.

Test the settings
```
dig -t mx dev.tarantool.org
dev.tarantool.org.	3600	IN	MX	100 dev.tarantool.org.
```

Make sure the SPF record has the IP address of the host
```
dev.tarantool.org.	TXT	v=spf1 ipv4:95.163.249.249 a mx ~all
```

The DMARC record should be like
```
_dmarc.dev.tarantool.org IN TXT “v=DMARC1; p=reject;”
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
At first we configure postfix without [STARTTLS](https://en.wikipedia.org/wiki/Opportunistic\_TLS).
Later we will enhance the configuration.

`main.cfg`
```
queue_directory = /var/spool/postfix
command_directory = /usr/sbin
daemon_directory = /usr/libexec/postfix
data_directory = /var/lib/postfix

mail_owner = postfix

myhostname = dev.tarantool.org
mydomain = dev.tarantool.org
myorigin = $mydomain
mydestination = $mydomain, localhost

inet_protocols = ipv4
inet_interfaces = $myhostname, localhost

unknown_local_recipient_reject_code = 550

mynetworks_style = host

alias_maps = hash:/etc/aliases
recipient_delimiter = +
owner_request_special = no

in_flow_delay = 1s
default_destination_rate_delay = 30s
default_destination_recipient_limit = 50

debug_peer_level = 2

sendmail_path = /usr/sbin/sendmail.postfix
newaliases_path = /usr/bin/newaliases.postfix
mailq_path = /usr/bin/mailq.postfix

setgid_group = postdrop

html_directory = no
manpage_directory = /usr/share/man

# Up to 1M
message_size_limit = 1048576

#smtpd_banner = ESMTP Postfix

smtp_use_tls = yes
smtp_tls_note_starttls_offer = yes
smtp_tls_protocols = !SSLv2, !SSLv3

smtpd_use_tls = yes
smtpd_tls_auth_only = yes
smtpd_tls_security_level = may
smtpd_tls_protocols = !SSLv2, !SSLv3
smtpd_tls_cert_file = /etc/letsencrypt/live/dev.tarantool.org/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/dev.tarantool.org/privkey.pem
smtpd_tls_loglevel = 1
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s
tls_random_source = dev:/dev/urandom

#transport_maps = hash:/etc/postfix/transport

disable_vrfy_command = yes
show_user_unknown_table_name = no
smtpd_helo_required = yes
smtpd_helo_restrictions=
	permit_mynetworks,
	permit_sasl_authenticated,
	reject_invalid_hostname,
	reject_non_fqdn_hostname,
	reject_invalid_helo_hostname,
	reject_unknown_helo_hostname

smtpd_sender_restrictions=
	reject_non_fqdn_sender,
	reject_unknown_sender_domain,
	reject_unlisted_sender,
	permit_mynetworks,
	permit_sasl_authenticated

smtpd_recipient_restrictions=
	reject_non_fqdn_recipient,
	reject_unknown_recipient_domain,
	reject_unlisted_recipient,
	permit_mynetworks,
	permit_sasl_authenticated,
	reject_unauth_destination
	reject_invalid_hostname

smtpd_data_restrictions=
	reject_unauth_pipelining,
	reject_multi_recipient_bounce

smtpd_etrn_restrictions=
	permit_mynetworks,
	permit_sasl_authenticated,
	reject

#smtpd_milters           = inet:127.0.0.1:8891
#non_smtpd_milters       = $smtpd_milters
#milter_default_action   = accept
#milter_protocol         = 2
```

`master.cfg`
```
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (yes)   (never) (100)
# ==========================================================================
smtp      inet  n       -       n       -       -       smtpd -v
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
#spammers may use it
#bounce    unix  -       -       n       -       0       bounce
bounce    unix  -       -       n       -       0       discard
#same here
#defer     unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       discard
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       n       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
```

Since we have certbot installed lets setup STARTTLS with
certificates obtained. Add the following to the `main.cfg`.
```
smtp_use_tls = yes
smtp_tls_note_starttls_offer = yes
smtp_tls_protocols = !SSLv2, !SSLv3

smtpd_use_tls = yes
smtpd_tls_auth_only = yes
smtpd_tls_security_level = may
smtpd_tls_protocols = !SSLv2, !SSLv3
smtpd_tls_cert_file = /etc/letsencrypt/live/dev.tarantool.org/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/dev.tarantool.org/privkey.pem
smtpd_tls_loglevel = 1
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s

tls_random_source = dev:/dev/urandom
```
