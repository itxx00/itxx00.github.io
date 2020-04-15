---
layout: post
title: "python脚本解压gbk编码zip"
description: ""
categories: [shell]
tags: [bash, python ]
---

> 编码问题很烦人

* Kramdown table of contents
{:toc .toc}

gbk编码的zip在linux下解压出来文件名会乱码，可以用下面脚本解压过程中转换下

```
#!/usr/bin/env python2
# coding: utf-8

import os
import sys
import zipfile

f = zipfile.ZipFile(sys.argv[1],"r");
for n in f.namelist():
    try:
        u = n.decode("gbk")
    except:
        u = n
    p = os.path.dirname(u)
    if not p:
        continue
    if not os.path.exists(p):
        os.makedirs(p)
    d = file.read(n)
    if os.path.exists(u):
        continue
    with open(u, "w") as o:
        o.write(data)
```
