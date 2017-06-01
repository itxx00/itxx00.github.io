---
layout: post
title: "shell脚本coding style总结"
description: ""
categories: [shell]
tags: [bash, coding_style ]
---

> 这里总结了个人比较推崇的shell脚本coding style，编写出方便阅读和维护的脚本是运维人员的基本操守。

* Kramdown table of contents
{:toc .toc}

1.关于缩进： 一个tab定义为4个空格；
关于这个缩进距离貌似有太多的的说法了，有的会用8个空格，有的用2个空格，还有的用4个空格；不过最终取决于团队风格。但是记住，正确的做法是项目、文件、成员间全部统一，千万不要出现一个项目内各种缩进，tab和空格混用，甚至一个文件中的缩进都不统一的情况。在vim中设置如下：
```
set ts=4
set sw=4
set expandtab
```

2.尽量缩短单行的长度，最好不超过72字符（注意：这个限制因历史原因导致，为了兼容那些老的终端设备而考虑，实际上现在已经不适用了，单行代码可以更长，只要不要超过大多数屏幕的输出宽度就好）

bad:

```
thisisaverylongline || thisisanotherlongline
```

good:

```
thisisaverylongline ||
    thisisanotherlongline
```

3.注释尽量保持清晰的层次,#号与注释文本间保持一个空格以示和代码的区分

bad:

```
#this is a comment
#this is a code line
```

good:

```
# this is a comment
#this is code line
```

4.定义函数可以不用写function关键字，函数名字尽量短小无歧义，尽量传递返回值

bad:

```
function  cmd1
```

good:

```
cmd_start() {
   dosomethinghere
   return 0
}
```

5.全局变量用大写，局部变量用小写，函数内尽量不要使用全局变量，以免混淆导致变量覆盖,注意尽量使用小写表示变量

bad:
```
foo() {
    a=2
    echo $a
}
a=1
echo $a
```

good:
```
foo(){
    local a
    a=2
    echo $a
}
A=1
echo $A
```

6.使用内建的[] 、[[]] 进行条件测试，避免使用test命令

bad:
```
test -f filename
```

good:
```
[ -f filename ]
[[ -n $string ]]
```

7.使用$(())进行普通运算，尽量避免使用expr或其他外部命令 $[]也可用于计算

bad:
```
num=$(expr 1 + 1)
```

good:
```
num=$((1+1))
num=$[1+2]
```

8.管道符左右都应加空格，重定向符空格加左不加右

bad:
```
find /data -name tmp*|xargs rm -fr
cat a>b
```

good:
```
find /data -name tmp* | xargs rm -fr
cat a >b
```
9.当`source`命令属于外部命令的时候，我们应尽量使用`.`   ，当视力不好怕写错看错的时候，应尽量不用`.`而用`source`，在实际使用中我们发现`.`号极难辨认，这里推荐使用source命令，更方便维护。

bad:
```
. func_file
```

good:
```
source func_file
```

10.if 和 then 之间使用分号+空格分隔，不要用换行，书写上和c style类似：

bad:
```
grep string logfile
if [ $? -ne 0 ]
then
  dosomething
fi

while 1
do
  do something here
done
```

good:

```
if grep string logfile; then
    dosomething
fi

while 1; do
   do something here
done
```
注意;和then间有一个空格的距离

11.如果grep能直接处理文件输入那就不要和cat连用; 如果wc能直接从文件统计就不要和cat连用; 如果grep能统计行数就不要和wc连用

bad:
```
cat file | grep tring
cat file.list | wc -l
grep avi av.list | wc -l
```

good:

```
grep tring file
wc -l file.list
grep -c avi av.list
```

12.如果awk的时候需要搜索恰好awk又能搜索，那么就不要再和grep连用

bad:
```
dosomething | grep tring | awk '{print $1}'
```

good:
```
dosomething | awk '/tring/' '{print $1}'
```

13、尽量编写兼容旧版本shell风格且含义清晰的代码，不推荐不兼容写法或者不方便他人维护的代码

bad:
```
do_some_thing |& do_another_thing
do_some_thing &>> some_where
```

good:
```
do_some_thing 2>&1 | do_another_thing
do_some_thing >some_where 2>&1
```

***注意*** : 以上仅代表个人习惯和理解，并不能适用于所有人，适合的才是最好的。

拓展阅读文档：
- http://kodango.me/simple-bash-programming-skills
- http://kodango.me/simple-bash-programming-skills-2
- http://www.commandlinefu.com/commands/browse
- http://wiki.bash-hackers.org/scripting/style
