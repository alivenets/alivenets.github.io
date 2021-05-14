---
layout: post
title:  "TIL 2021-02-28"
date:   2021-02-28 22:00:00 +0200
categories: til learned c++
---

The digest of "Today I learned" topics.

# C++ and alternative operators

It turns out that C++ supports alternative operators having C legacy ([reference](https://en.cppreference.com/w/cpp/language/operator_alternative)). This may bring some discomfort during compilation if `<iso646.h>` was somehow included explicitly or implicitly.
However, on the compiler level we can disable it via `-fno-operator-names`.

# Python cross-compilation

For python disttools, in order to cross-compile c extensions, one needs to have cross-compiled version of python too.