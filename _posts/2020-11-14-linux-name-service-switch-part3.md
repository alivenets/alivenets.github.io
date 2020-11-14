---
layout: post
title:  "Linux Name Service Switch - Part 3 - External user database service"
date:   2020-11-14 22:00:00 +0200
categories: glibc linux nss nss-plugin dbus dynamic-group dynamic-user
---

The 3rd part is dedicated to the communication with external service providing names for NSS.

# UserDB as external service

Now, let's extract static data from NSS plugin and move it to external service. That would that the data can be dynamically changed at runtime independently of plugin.

## IPC with UserDB service

First idea is to use the same D-Bus IPC to communicate with external service, which is already used to communicate with D-Bus Echo service. However, classical D-Bus communication via `dbus-daemon` will not work. The reason is that `dbus-daemon` at authentication step implicitly calls NSS to resolve names and groups for the connection to external name service (See [D-Bus specification](https://dbus.freedesktop.org/doc/dbus-specification.html#auth-command-auth)) So, this leads to deadlock in the NSS plugin.

To work around this issue, we are using D-Bus in P2P mode. However, as for now, `sdbus-c++` does not support P2P connections, so, we fall back to `Glibmm` for `userdb-service`.

# UserDB service API

UserDB service provides quite simple, yet fulfilling our needs API. Here is the introspection file.

```xml
<node>
    <interface name="com.example.UserDb">
        <method name="ListGroups">
            <arg type="as" name="groups" direction="out"/>
        </method>

        <method name="ListUsers">
            <arg type="as" name="users" direction="out"/>
        </method>

        <method name="GetUserByName">
            <arg type="s" name="name" direction="in"/>  
            <arg type="u" name="uid" direction="out"/>
            <arg type="u" name="gid" direction="out" />
        </method>

        <method name="GetUserById">
            <arg type="u" name="uid" direction="in"/>
            <arg type="s" name="name" direction="out"/>
            <arg type="u" name="gid" direction="out" />
        </method>

        <method name="GetGroupByName">
            <arg type="s" name="name" direction="in"/>
            <arg type="u" name="gid" direction="out"/>
            <arg type="as" name="members" direction="out"/>
        </method>
        
        <method name="GetGroupById">
            <arg type="u" name="gid" direction="in"/>
            <arg type="s" name="name" direction="out"/>
            <arg type="as" name="members" direction="out"/>
        </method>  
    </interface>
</node>
```

Here, we only need the names and IDs of users and groups, so the interface becomes much simpler.

In our implementation, UserDB will return information about users `com_example_dynamicuser` and `com_example_dynamicuser2` and main groups for them

Note, that `service-client` group is already registered in `/etc/group` file, main NSS database. `GetGroupByName`/`GetGroupById` may however return this group once more with extended memberships, which indeed allows us to utilize this external user database.

The straightforward implementation of UserDB service is [here](https://github.com/alivenets/nss-plugin-example/blob/master/userdb-service/main.cpp).

# UserDB client

The UserDB client implementation is pretty straightforward as well. For our case, to simplify integration into NSS plugin, special [UserDB test client](https://github.com/alivenets/nss-plugin-example/blob/master/userdb-client/main.c) in C was written.

# NSS plugin refactoring

Now, stepping into main part - NSS plugin library. 
The main change here, that for the group iteration we use API.

The workflow is the following:
`setgrent` - list all groups, store it in the array, reset iterator
`getgrent_r` - get next group name from the list, get group information by name, obtain membership from UserDB service
`endgrent` - reset iterator

Here is the sequence diagram of main workflow of some user app, `id someuser` as an example.

![](/public/assets/nss-userdb-sequence-diagram.png)

# Testing

The workflow testing extends [this procedure](https://alivenets.github.io/glibc/linux/nss/nss-plugin/dbus/dynamic-group/dynamic-user/2020/09/22/linux-name-service-switch-part2.html#testing-access-workflow)

1. Build and install NSS plugin 

2. Run services

  1.  **Run UserDB service**:
```
    userdb-service/userdb-service &
```

  2. Run `dbus-service`: 
```
    dbus-service/dbus-service &
```

3. Call D-Bus  without file `/tmp/enable-dynamic-group`:
```
sudo -u com_example_dynamicuser dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```

Here, we get `org.freedesktop.DBus.Error.AccessDenied`.

4. Create `/tmp/enable-dynamic-group` file and reload D-Bus daemon cache:

```
dbus-send --system --print-reply --dest=org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus.ReloadConfig
sudo -u com_example_dynamicuser dbus-send --system --print-reply --dest=com.example.EchoService /com/example/EchoService com.example.EchoService.Echo string:"abc"
```

The resulting code is [here](https://github.com/alivenets/nss-plugin-example/commit/53d408ff64dc7015e996a4b8b43c0a65338838f7).

# Potential improvements

The code can be improved in several ways:
* Performance optimizations
  * In NSS configuration, go to our plugin only if user is not found in other plugins.
  * Implement data caching in the NSS plugin
  * Using another IPC instead of D-Bus, since D-Bus may appear to be slow

