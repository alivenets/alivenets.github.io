---
layout: post
title:  "Linux Name Service Switch - Part 1"
date:   2020-09-05 00:00:00 +0200
categories: glibc linux nss nss-extension
---

NSS (Name Service Switch), [Documetation](https://www.gnu.org/software/libc/manual/html_node/Name-Service-Switch.html) is a part of Linux name resolution system, also being extendable. When any application requests the information about users and groups via Glibc functions `getpwnam`, `getgrnam`, glibc goes to NSS resolution module.

The traditional way to work with users, groups and hosts was previously `/etc/passwd`, `/etc/group` and `/etc/host` accordingly (not counting the `/etc/shadow` with true user passwords). Then, when the need to extend database with Network Information Services appeared, NSS has been designed.

The greatness of NSS is that the system is extendable with plugins. So, the developer may write a NSS plugin providing additional user names in runtime frmo static configuration or from respons of another name service. To extend the NSS, C library may be provided without changing the Glibc code at all. There is specified API to write exported library functions: [NSS modules interface](https://www.gnu.org/software/libc/manual/html_node/NSS-Modules-Interface.html), [NSS modules internals](https://www.gnu.org/software/libc/manual/html_node/NSS-Module-Function-Internals.html)

The sequence diagram shows the example how the user information is fetched from name services (default Linux configuration with `/etc/passwd` and `systemd`)

{% plantuml %}
App -> Glibc : getpwnam_r
activate Glibc
Glibc -> NSS
activate NSS
NSS -> NSS : Iterate over plugins
NSS -> passwd : Request info from passwd file
passwd -> NSS : pwd struct
NSS -> systemd : Send request to external SystemD UserDB service
systemd -> NSS : pwd struct
return pwd struct
return pwd struct
{% endplantuml %}

# Basics

The extract from NSS tutorial provides different types of names resolved in GlibC
 * **aliases** - Mail aliases
 * **ethers** - Ethernet numbers,
 * **group** - Groups of users, see Group Database.
 * **gshadow** - Group passphrase hashes and related information.
 * **hosts** - Host names and numbers, see Host Names.
 * **initgroups** - Supplementary group access list.
 * **netgroup** - Network wide list of host and users, see Netgroup Database.
 * **networks** - Network names and numbers, see Networks Database.
 * **passwd** - User identities, see User Database.
 * **protocols** - Network protocols, see Protocols Database.
 * **publickey** - Public keys for Secure RPC.
 * **rpc** - Remote procedure call names and numbers.
 * **services** - Network services, see Services Database.
 * **shadow** - User passphrase hashes and related information.

# Tracing NSS

To get the NSS entities, `getent` command can be used to iterate through all entities from name services.

```bash
$ getent passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
vagrant:x:1000:1000:vagrant,,,:/home/vagrant:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
root:x:0:0:root:/root:/bin/sh
nobody:x:65534:65534:nobody:/:/usr/sbin/nologin
#...
```

```bash
$ getent group
root:x:0:
daemon:x:1:
systemd-coredump:x:999:
vboxsf:x:998:
root:x:0:
nogroup:x:65534:
#...
```

Additionally, we can request specific service using `-s`

```
$ getent -s systemd passwd
root:x:0:0:root:/root:/bin/sh
nobody:x:65534:65534:nobody:/:/usr/sbin/nologin
```

# NSS Extension Examples

## SystemD

SystemD utilizes NSS in order to provide `DynamicUser` functionality [Lennart Poettering's blog](http://0pointer.net/blog/dynamic-users-with-systemd.html), [manual page](https://www.freedesktop.org/software/systemd/man/nss-systemd.html). The code is here: [Github link](https://github.com/systemd/systemd/tree/master/src/nss-systemd)

## LDAP

LDAP also provides NSS plugin to communicate with LDAP name service: [Wiki](https://wiki.debian.org/LDAP/NSS). The code is here: [Github link](https://github.com/arthurdejong/nss-pam-ldapd)

## Docker

There are several NSS extensions to provide host names from Docker service: [Github 1](https://github.com/dex4er/nss-docker), [Github 2](https://github.com/danni/docker-nss).
