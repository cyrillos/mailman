Installing OS
=============

As a base OS we use centos-7.5 series.

The minimum VM parameters are:

 - 40G of disk space
 - 2G of memory

Configure OS environment
------------------------

Before continue lets assume we've the following OS environment parameters:

 - Hostname: mailman.tarantool.org
 - IP: 95.163.249.249

[//]: # The domain should be already registered in DNS and properly resolved.
[//]: # Thus there should be at least the records
[//]: # 
[//]: # ```
[//]: # @	3600	 IN 	A	198.137.202.1
[//]: # @	3600	 IN 	MX	100	ml.tarantool.org.
[//]: # ```

Generate SSH key and authorize it (these actions are to be done on your
_client_ machine from which you are planning to access the server and
manage it!)
```
ssh-keygen -f ~/.ssh/devel_tarantool_id
ssh-copy-id -i ~/.ssh/devel_tarantool_id root@tarantool.org
```

Make sure the `/etc/ssh/sshd_config` has the following:
```
PermitRootLogin yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
UsePAM yes
X11Forwarding no
```

And restart the SSH service
```
systemctl restart sshd
```

Install firewalld daemon
```
yum install firewalld
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --permanent --zone=public --add-interface=eth0
firewall-cmd --reload
```

Make sure only a few ports are opened on public interface
```
firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources: 
  services: ssh dhcpv6-client
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

[//]: # Install automatic updates for critical security fixes
[//]: # ```
[//]: # dnf install -y dnf-automatic
[//]: # ```
[//]: # 
[//]: # In `/etc/dnf/automatic.conf` set
[//]: # ```
[//]: # [commands]
[//]: # upgrade_type = security
[//]: # download_updates = yes
[//]: # apply_updates = yes
[//]: # [emitters]
[//]: # emit_via = None
[//]: # ```
[//]: # 
[//]: # And enable it
[//]: # ```
[//]: # systemctl enable --now dnf-automatic.timer
[//]: # ```
