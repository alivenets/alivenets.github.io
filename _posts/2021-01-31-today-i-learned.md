---
layout: post
title:  "TIL 2021-01-31"
date:   2021-01-31 22:00:00 +0200
categories: til learned pytest bash glib dbus coverity cmake docker python asan libc
---

The digest of "Today I learned" topics.

# Pytest fixture dependencies

Fixtures can depend on other fixtures. The example is here:

```python
@pytest.fixture
def fixture1():
    yield 42

@pytest.fixture
def fixture2(fixture1):
    yield fixture1 / 2
```

# Bash, conditions and quote handling

In Bash, when using operator `[`, when executing command inside, quotes are required:

```bash
if [ "$(id -un)" = "0" ]; then
    echo "I am root"
fi
```

# Glib, D-Bus sockets and FD_CLOEXEC

In Glib, D-Bus sockets have `FD_CLOEXEC` option by default. [Code reference](https://github.com/GNOME/glib/blob/master/gio/gsocket.c#L623).

# Coverity, emit DB and virtual machine

Apparently, coverity cannot use working directory with `vboxsf` file system, where emit DB has to be created and locked. Since coverity is trying to lock the database, this fails in Virtualbox if using directory on `vboxsf` partition. So, any other directory should be used on guest-only filesystem.

# CMake and include directories

Advice: when using `include_directories`/`target_include_directories`, do not forget to add `SYSTEM` for libraries and directories that should be considered as system ones. This will make static code analyzers ignore errors when parsing source files.

# Logs and testing

Sometimes, when testing services, there may be no API providing the result. Meaning, the service may be the black box executing something and reporting failure/success to logs. Therefore, it should be possible to write tests using service logs only. That means: logs should sufficient enough to analyze the failure cause.

# `getgrgid_r`/`getgrnam_r` and buffer size

As from `man getgrnam_r`, it is written that the buffer size should be requested via `sysconf(_SC_GETGR_R_SIZE_MAX)`. However, reading manual thoroughly, it `returns either -1, without changing errno, or an initial suggested size for buf.  (If this size is too small, the call fails with ERANGE, in which case the caller can retry with a larger buffer.)`. Next, looking inside Glibc, for Linux(POSIX) case, [here](https://code.woboq.org/userspace/glibc/sysdeps/posix/sysconf.c.html#537), sysconf returns `NSS_BUFLEN_GROUP` == 1024,[code](https://code.woboq.org/userspace/glibc/grp/grp.h.html#114).

# Dockerfile conditionals

It is possible to write conditional operation in Dockerfile, so they execute depending on the environment variable:

```
RUN if [ "$COND_VARIABLE" == "1" ]; then \
       echo "something" > /tmp/newfile \ 
    fi
```

# Missing stacktraces in LSAN

Sometimes, when running LSAN and leak happens in external libraries, the stack trace is printed incomplete or bogus. The solution would be to run with `LSAN_OPTIONS=fast_unwind_on_malloc=0`, added to existing `LSAN_OPTIONS`. [Source](https://github.com/google/sanitizers/issues/870).

# Python Popen

`subprocess.Popen` for Python3 ALWAYS returns non-null `Popen` object.
