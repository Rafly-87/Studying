
# initial setup

## configure file system kernel module

## pattern

```
blacklist [nama_module]
install [nama_module] /bin/false
```

## object

| no  | object        |
| --- | ------------- |
| 1   | cramfs        |
| 2   | freevxfs      |
| 3   | hfs           |
| 4   | hfsplus       |
| 5   | jffs2         |
| 6   | overlay       |
| 7   | squasfs       |
| 8   | udf           |
| 9   | fireware-core |
| 10  | usb storage   |

## configure file system partition


## patern

```
[device] [mount_target] [mount_option] 0 0
```

## object

| no  | mount path     | mount option        |
| --- | -------------- | ------------------- |
| 1   | /tmp           | nodev,nosuid,noexec |
| 2   | /dev/shm       | nodev,nosuid,noexec |
| 3   | /home          | nodev,nosuid        |
| 4   | /var           | nodev,nosuid        |
| 5   | /var/tmp       | nodev,nosuid,noexec |
| 6   | /var/log       | nodev,nosuid,noexec |
| 7   | /var/log/audit | nodev,nosuid,noexec |


## package management

regularly update arch linux
```
pacman -Syu
```


## addidtional hardening



| name                   | path                                  | value                      |
| ---------------------- | ------------------------------------- | -------------------------- |
| core                   | /etc/security/limits.d/60-limits.conf | * hard core 0              |
| fs.protected_hardlinks | /etc/sysctl.d/fs-prottected.conf      | fs.protected_hardlinks = 1 |
| fs.protected_symlinks  | /etc/sysctl.d/fs-symlink.conf         | s.protected_symlinks = 1   |
| fs.suid_dumpable       | /etc/sysctl.d/fs-dumpable.conf        | fs.suid_dumpable = 0       |
| kernel.dmesg_restrict  | /etc/sysctl.d/dmesg_restrict.conf     | kernel.dmesg_restrict = 1  |
|                        |                                       |                            |


# services

## patern

```
systemctl stop [service_name]
```

```
systemctl mask [servicename]
```

## object

| no  | service               |
| --- | --------------------- |
| 1   | autofs.service        |
| 2   | avahi-daemon.socket   |
| 3   | avahi-daemon.service  |
| 4   | cockpit.socket        |
| 5   | cockpit.service       |
| 6   | kea-dhcp-ddns.service |
| 7   | kea-dhcp4.service     |
| 8   | kea-dhcp6.service     |

## configure client server

```
pacman -Rs ftp ldap telnet tftp
```

# network


## configure network kernel module

## pattern

```
blacklist [nama_module]
install [nama_module] /bin/false
```

## object

| no  | object        |
| --- | ------------- |
| 1   | atm           |
| 2   | can           |
| 3   | dccp          |
| 4   | tipc          |
| 5   | rds           |
| 6   | sctp          |

## Network Kernel Parameters

### ipv4

| name                              | path                               | value                                 |
| --------------------------------- | ---------------------------------- | ------------------------------------- |
| net.ipv4.ip_forward               | /etc/sysctl.d/ipv4.ip_forward.conf | net.ipv4.ip_forward = 0               |
| net.ipv4.conf.all.forwarding      | /etc/sysctl.d/                     | net.ipv4.conf.all.forwarding = 0      |
| net.ipv4.conf.default.forwarding  | /etc/sysctl.d/                     | net.ipv4.conf.default.forwarding = 0  |
| net.ipv4.conf.all.accept_redirect | /etc/sysctl.d/                     | net.ipv4.conf.all.accept_redirect = 0 |
|                                   |                                    |                                       |

### ipv6



# firewall


```
pacman -S firewalld
```

```
pacman -S nftables
```


```
systemctl enable firewalld
```

# access

| access    | alphabet | numeric |
| --------- | -------- | ------- |
| write     | w        | 4       |
| read      | r        | 2       |
| execution | x        | 1       |

chmod 0600 /etc/ssh/sshd_config

agoy.txt

chmod 2 agoy.txt

pemilik file = write and read
group 4 = write
other = write

```
drwxr-xr-x  2 agoy agoy   4096 Jul  3 01:43 ssh_config.d
```

```
drwx------  2 agoy agoy   4096 Jul  3 01:43 ssh_config.d
```

```
d rwx r-- ---  2 agoy agoy   4096 Jul  3 01:43 ssh_config.d
```
d = 


## patern

```
chown [user]:[group] [file_path]
```

```
cmod [modoption] [file_path]
```

## object


| user | group | modoption | file_path            |
| ---- | ----- | --------- | -------------------- |
| root | root  | 0600      | /etc/ssh/sshd_config |

# logging and auditing

