---
layout: post
title:  "TIL 2021-05-14"
date:   2021-05-14 16:00:00 +0200
categories: til learned c++ testing logging project-management linux ltrace json
---

The digest of "Today I learned" topics.

# Test data generation in runtime

For the unit testing, sometimes there is a need to have some test data set. It may be a bunch of JSON/XML files or a certificates.

There may be several approaches to the data generation:

1. Precreate files and commmit them to the repository with the generation script
2. Create the files during build ising generation script
3. Create files during tests

3rd way is not considered good because some generated files are not generated in a stable way,
particularly, certificates. Meaning, every run of 'openssl` gives you new certificates with new random numbers.

Ways 1 and 2 are more preferred.

# Proper logging approach - logging abstraction

When the project or component is just created and the development is ramping up, it is essential 
to create a logger abstraction to define component-internal functions and macros to abstract the logging
ffrom the logging library being used.

Reasoning behind: as long as component development and evolution goes, there may be a need to replace
the logging library being used. Therefore, having even macro wrappers would be much better,
since they allow to replace library calls easily.

# Big projects and feature freeze

Feature freeze (the state when new features are not allowed to be merged, except crucial ones and bugfixes)
may sound a good idea, if we need to relesae. However it brings some consequences:
 * the new changes pile up quickly so that next period after feature freeze will be a mess to merge new feature which would bring more bugs
 * the continous integration concept violations - we don't take the latest master

# Hint: ltrace

Don't forget to use ltrace if you want to see which calls to which shared object your Linux program is making. 

# Ordered JSON

There may be a case when standard JSON is not enough. For example, when parsing/serializing JSON, dictionaries cannot guarantee the order, meaning the serialized JSON is not equal to the original string.
To overcome this issue, we may use ordered JSON. Example is [here](https://github.com/alivenets/sandbox/tree/master/Cpp/OrderedJson).

