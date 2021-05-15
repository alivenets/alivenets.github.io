---
layout: post
title:  "TIL 2021-05-15"
date:   2021-05-15 18:00:00 +0200
categories: til learned c++ yocto oeqa systemd boost asio embedded monorepo
---

The digest of "Today I learned" topics.

# Variant handling

Variant handling is better to be done in runtime. Approach: variant coding (variant config) describing presence of some features,
signed with manufacturer private key.
Cannot be updated, only new one can be reflashed.

# C++ private status functions

Private static funxtions are most probably an indicator that the code is badly organized. The way to solve them - extract to the utility namespace.

# Useful C+=

* Boost.Asio signal_set - allows to handle Unix signals asynchronously.

# Yocto OEQA test output

Test output should be filtered for control characters or written to file to target FS, not being printed.

# C++ assignment operator overloading

For operators like `/=`, `*=` etc. it is easy to combine assignmen with appropriate arithmetic operator overloading.

Example:

```cpp
class Url
{
Url & operator/= (const std::string &s)
{
    *this = *this / s;
    return *this;
}

Url &operator/ (const std::string &s)
{
    this->s += "/" + s;
    return *this;
}

private:
    std::string s;
};
```

# Yocto service files
In Yocto, when installing recipes with SystemD services, sometimes for testing it is needed to reconfigure the service to run it in service mode.
For that purpose, environment variables in combinatiion with `EnvironmentFile` option rather be used. 
The reasoning behind is that installing and maintaining additional service file is cumbersome. Also, original and testing service file may be in conflict.
Additional option to extend the orignal service would be service drop-in file, but it may also be confusing for the developer.
Moreover, `EnvironmentFile` can serve not only for automated but also for manual testing.

# thoughts on Monorepo

Why is monorepo considered bad or good? From my work I can bring some reasoning to it.

Advantages:
 * easier testing
 * All code visible in one place
 * Easy to search
 * Common approach for everyting
 * Works good for small projects, web portals

Disadvantages:
 * Harder maintenans. Commits can break unrelated tests
 * Harder review
 * Longer CI runs. Typically, one has to build EVERYTHING to run tests
 * Longer manual testing. Again, one needs to build EVERYTHING
 * Large code base, slower Git operations
 * Not flexible enough if the system is heterogeneous, meaning a lot of components with different purpose,
   different code sources, or there may be even binaries or static libraries to be compiled.

So, for big projects it is better to split the product into components and cover it properly with tests before it is integrated. 
Thus, we avoid big pile of features being blocked by broken builds.
