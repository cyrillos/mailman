Installing OS
=============

As a base OS we use centos-7.5 series.

The minimum VM parameters are:

 - 40G of disk space
 - 2G of memory

Configure OS environment
------------------------

Before continue lets assume we've the following OS environment parameters:

 - Hostname: dev.tarantool.org
 - IP: 95.163.249.249

Asjust DNS records if they were not yet.
```
dev A 95.163.249.249
```

If the name is undefined yet due to fresh setup of the machine use `hostnamectl`
```
hostnamectl set-hostname dev.tarantool.org
```

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
