DNS records
===========

Summary DNS records should be like
```
dev     A           95.163.249.249
lists   A           95.163.249.249
dev	IN MX 100   dev.tarantool.org.
dev     TXT         v=spf1 a mx ip4:95.163.249.249 include:_spf.google.com ~all
_dmarc.dev.tarantool.org    TXT			v=DMARC1; p=none
```
