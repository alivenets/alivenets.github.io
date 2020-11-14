---
layout: post
title:  "Linux Name Service Switch - Part 2 - D-Bus and NSS"
date:   2020-09-22 12:00:00 +0200
categories: glibc linux nss nss-plugin dbus dynamic-group dynamic-user
---

Here we continue to talk about NSS and how we can use and extend it for our purposes.

# Problem statement

Imagine that we have the following restrictions:

* There must be D-Bus service
* D-Bus service has policy to allow only group members to use the interface
* The user is added dynamically, that means that there may be no user in the system, and any user can be created on the fly or be provided by other name services, like LDAP.

## D-Bus and user policy

Let's look into the sample code [here](https://github.com/alivenets/nss-plugin-example/commit/908cd4d55641134368991952aeaded21c9a52b1d)

Here is the demonstration how we can access the interface. First, let's run our D-Bus Echo service:
```
dbus-service/dbus-service &
```

Using `busctl`, we can check now if the `EchoService` is provided.

Then, call Echo function of the service from the current logged user.

```
dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```

This does not work, showing `org.freedesktop.DBus.Error.AccessDenied` error. while

However,

```
sudo -u service-user dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```
works and returns `"abc"` string, because the user is already in `service-client` group. 

## D-Bus authorization internals

In the code of [D-Bus authentication](https://gitlab.freedesktop.org/dbus/dbus/-/blob/master/bus/connection.c#L417),`dbus-daemon` reads the credentials  t the authorization stage. In details, the process is happening (here)[https://gitlab.freedesktop.org/dbus/dbus/-/blob/master/bus/connection.c#L417}. According to the protocol, at the authorization stage, client sends UID, then the server reads the user credentials from the system using Glibc and NSS.

## D-Bus daemon cache

From the `dbus-daemon` implementation, even if NSS can provide some data dynamically depending on system state, `dbus-daemon` caches it in the internal database.

Luckily, `dbus-daemon` service interface has special method `org.freedesktop.DBus.ReloadConfig` which clears cache and allows to update user information on the next D-Bus request.

# Writing our own NSS plugin

The NSS plugin must have predefined interfaces, as described in [Glibc manual](https://www.gnu.org/software/libc/manual/html_node/Name-Service-Switch.html).

Let's analyze the first iteration approach to the plugin:

* The NSS plugin provides some static data
* NSS plugin provides new user entities and group memberships
* New user entities are assigned to existing groups defined in `/etc/group`. That is indeed the extension of existing group membership in runtime
* The plugin is reconfigured checking if the file is present on file system (straightforward way)
* The plugin uses pthread mutex for entity iterator to block while iterating through groups. That behavior is required due to not-so-reentrant behavior of `getpwent_r`/`getgrent_r`

As an inspiration, `nss-systemd` plugin was used: [link](https://github.com/systemd/systemd/tree/master/src/nss-systemd).
The plugin implementation is available [here](ttps://github.com/alivenets/nss-plugin-example/commit/cce6aff6219a093ba6a7a63c42f6a9277494a095).

The implementation provides `com_example_dynamicuser` user. This user can be conditionally added to `service-client` group if file `/tmp/enable-dynamic-group` exists.

# Testing access workflow

1. Build and install NSS plugin
2. Run `dbus-service`:
```
    dbus-service/dbus-service &
```

3. Call D-Bus  without file `/tmp/enable-dynamic-group`:
```
sudo -u com_example_dynamicuser dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```

Here, we get `org.freedesktop.DBus.Error.AccessDenied`.

4. Create `/tmp/enable-dynamic-group` file,  however, now, we have to reload daemon cache to clean cached user supplementary groups information.

```
dbus-send --system --print-reply --dest=org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus.ReloadConfig
sudo -u com_example_dynamicuser dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```

Success! Now we get the data from `EchoService` after dynamically assigning the dynamic user to the dynamic group.
