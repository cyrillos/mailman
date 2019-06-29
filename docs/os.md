Installing OS
=============

As a base OS we use [Fedora 30](https://getfedora.org/en/workstation/download/) netinstall image.
Just go to the site and download an appropriate iso file.

The minimum VM parameters are:

 - 40G of disk space
 - 4G of memory

Choose "Minimal install" when selecting software during setup wizard. We will
install required packages later via command line. Don't forget to setup strong
password for the `root` user.

Configure OS environment
------------------------

Before continue lets assume we've the following OS environment parameters:

 - Hostname: tarantool.org
 - IP: 198.137.202.1

The domain should be already registered in DNS and properly resolved.

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

Make sure only a few ports are opened on public interface
```
firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: dhcpv6-client mdns ssh
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```
