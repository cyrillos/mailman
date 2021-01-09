Installing public-inbox
=======================

There is no prebuilt [public-inbox](https://public-inbox.org/) package
so we need to fetch the repo and setup it by hands. See the
[installation](https://public-inbox.org/INSTALL.html) manual.

The mail archive for Mailman mails gonna be saved in `/home/pi`
directory. For this sake create a user

```
useradd -m pi
```

We gonna use `public-inbox-watch` to fetch new mails because there
is no easy integration with old Mailman-2.x series. For this sake
we should force subscribe our user to mailman lists (usually via
administrative interface).

It is more convenient for `public-inbox-watch` to operate with
maildir archives. For this sake update `postfix` configuration
in `main.cf`
```
#
# For public-inbox sake
home_mailbox = Maildir/
```

and restart the postfix.

Note that mailman may send password reminders so make sure
this feature is turned off.

Create services we need to handle web access and a watcher.

The socket service `/etc/systemd/system/public-inbox-httpd.socket`
```
[Unit]
Description = public-inbox-httpd socket

[Socket]
ListenStream = 8080
Service = public-inbox-httpd@1.service

[Install]
WantedBy = sockets.target
```

The httpd handler `/etc/systemd/system/public-inbox-httpd@1.service`
```
[Unit]
Description = public-inbox PSGI server %i
Wants = public-inbox-httpd.socket
After = public-inbox-httpd.socket

[Service]
Environment = PI_CONFIG=/home/pi/.public-inbox/config \
PATH=/usr/local/bin:/usr/bin:/bin \
#PERL_INLINE_DIRECTORY=/tmp/.pub-inline

LimitNOFILE = 30000
#ExecStartPre = /usr/bin/mkdir -p -m 1777 /tmp/.pub-inline
ExecStart = /usr/local/bin/public-inbox-httpd \
-1 /var/log/public-inbox/httpd.out.log
StandardError = syslog

# NonBlocking is REQUIRED to avoid a race condition if running
# simultaneous services
NonBlocking = true
Sockets = public-inbox-httpd.socket

KillSignal = SIGQUIT
User = pi
Group = pi
ExecReload = /bin/kill -HUP $MAINPID
TimeoutStopSec = 86400
KillMode = process

[Install]
WantedBy = multi-user.target
```

Enable it
```
systemctl enable public-inbox-httpd@1.service
```

To make nginx connect to `public-inbox-httpd` service update `nginx.conf`
```
location ~* ^/tarantool-patches(.*)$ {
    proxy_set_header    HOST $host;
    proxy_set_header    X-Real-IP $remote_addr;
    proxy_set_header    X-Forwarded-Proto $scheme;
    proxy_pass          http://127.0.0.1:8080$request_uri;
}
location ~* ^/tarantool-discussions(.*)$ {
    proxy_set_header    HOST $host;
    proxy_set_header    X-Real-IP $remote_addr;
    proxy_set_header    X-Forwarded-Proto $scheme;
    proxy_pass          http://127.0.0.1:8080$request_uri;
}
```

The mail archives are stored in `/home/pi/tarantool-discussions.git`
and `/home/pi/tarantool-patches.git` repos. Since we are subscribed
to mailing list updates the mailman will post new messages to postfix
which then save them in `/home/pi/Maildir`.

Put the following into `/home/pi/.public-inbox/config`
```
[publicinbox "tarantool-patches"]
        address = tarantool-patches@dev.tarantool.org.
        url = https://127.0.0.1:8080/
        inboxdir = /home/pi/tarantool-patches.git
        indexlevel = full
        spamcheck = none
        watch = maildir:/home/pi/Maildir/
        watchheader = List-Id:<tarantool-patches.dev.tarantool.org>
[publicinbox "tarantool-discussions"]
        address = tarantool-discussions@dev.tarantool.org.
        #url = https://lists.tarantool.org/tarantool-discussions
        url = https://127.0.0.1:8080/
        inboxdir = /home/pi/tarantool-discussions.git
        indexlevel = full
        spamcheck = none
        watch = maildir:/home/pi/Maildir/
        watchheader = List-Id:<tarantool-discussions.dev.tarantool.org>
```
