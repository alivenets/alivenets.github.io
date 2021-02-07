---
layout: post
title:  "TIL 2021-02-07"
date:   2021-02-07 22:00:00 +0200
categories: til learned cmake json vagrant
---

The digest of "Today I learned" topics.

# CMake and JSON parser

In CMake, as of version 3.19, JSON parsing functionality has been added:
[here](https://cmake.org/cmake/help/v3.19/command/string.html#json).
For CMake <= 3.19, [this library](https://github.com/sbellus/json-cmake) can be used.

# Vagrant and load order

When Vagrant VM is created, the files are merged in the following order:
 * Vagrantfile packaged with the box
 * Vagrantfile from the configuration directory for the user, usually `~/.vagrant.d`
 * Vagrantfile in the project tree

So, it is possible to define generic configuration, and then inherit it in project Vagrantfile.

[Source 1](https://www.vagrantup.com/docs/vagrantfile#load-order), [Source 2](https://mgdm.net/weblog/vagrantfile-inheritance/)

# Vagrant and private box registry authentication

New option `config.vm.box_download_options` has been added in Vagrant 2.2.8,
which allows to pass additional options to box downloader ('curl'):
[here](https://www.vagrantup.com/docs/vagrantfile/machine_settings#config-vm-box_download_options).

So, to make it simple, place `.curlrc` in your home directory with the following contents:

```
cert = "/path/to/client-crt"
key = "/path/to/key"
netrc = "~/.netrc"
```

and `.netrc`:

```
machine <registry_url>
user <user_name>
password <registry_password>
```

then add the reference to `.curlrc` file in Vagrantfile:

```ruby
config.vm.box_download_options = {config: "/path/to/curlrc"}
```
