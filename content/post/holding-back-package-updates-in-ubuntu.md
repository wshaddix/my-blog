---
title: "Holding Back Package Updates in Ubuntu"
date: 2018-05-21T17:36:42-04:00
lastmod: 2018-05-21T17:36:42-04:00
draft: false
keywords: []
description: "Holding Back Package Updates in Ubuntu"
tags: ["ubuntu"]
categories: ["ubuntu"]

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---
How to hold back packages from being updated in Ubuntu
<!--more-->

# Overview

If you want to hold back a package from being upgrade in ubuntu you can use the `apt-mark` command.

## Hold back a package

This will prevent the package from being updated when you upgrade.

```bash
sudo apt-mark hold <package name>
```

## UnHold a package

This will unhold the package and make it upgrade automatically when you upgrade.

```bash
sudo apt-mark unhold <package name>
```

## Show which packages are held back

If you're like me and you forget which packages you held back you can show a list of all held packages.

```bash
apt-mark showhold
```