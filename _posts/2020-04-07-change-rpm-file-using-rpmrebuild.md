---
layout: post
title: "使用rpmrebuild修改rpm包内容"
description: "一种rpm包的魔改方式"
categories: [shell]
tags: [bash,rpm, rpmrebuild ]
---

> 某些特殊紧急情况下... ...


* Kramdown table of contents
{:toc .toc}

某些特殊紧急情况下没法等到重新从源码编译打包，手里只有一个打包好的rpm，但是里面内容需要在安装前就改掉，比如修改某个文件内容等，这个时候rpmrebuild命令可以派上用场。
rpmrebuild工作时会把rpm包内容释放到一个临时目录，如果需要修改rpm包里面的文件的话， 可以通过-m参数指定执行的命令，比如/bin/bash，这样就可以得到一个交互式的shell，
有了交互式shell想象空间就很大了，你可以在这个shell环境下对rpm包释放出来的文件任意修改，当退出这个shell时，rpmrebuild会把改动打包回新的rpm。
例如：
```
rpmrebuild -m /bin/bash -np rpm/xxx.rpm
# 此时我们得到一个交互shell，
# 比如知道需要修改的文件名为aaa，可以这样操作：
find / -name aaa
# 尽情发挥吧，完了退出
ctrl+D
```
现在你就得到修改好内容之后的新rpm了。
