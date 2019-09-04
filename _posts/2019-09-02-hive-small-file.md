---
layout: post
title: "标题"
description: "描述"
categories: [shell]
tags: [bash, ]
---

> 介绍hie小文件常见处理方法

* Kramdown table of contents
{:toc .toc}

## hive的文件产生过程


## 小文件太多的影响


## 为什么会产生小文件

## 如何处理小文件

### case 1
```
INSERT OVERWRITE TABLE tb1
    SELECT * FROM tb2
ORDER BY 1;
ALTER TABLE tb2 RENAME TO b_tb2;
ALTER TABLE tb1 RENAME TO tb2;

```
### case 2
```
INSERT TABLE tb1
SELECT c1, c2 FROM (
    SELECT c1, c2
    FROM tb2
    WHERE xxx
      AND xxx
) t
ORDER BY c1, c2;
```

### case 3
```
SELECT c1
FROM  (
    xxx
) t
GROUP BY x;
```

### case 4
```
INSERT OVERWRITE TABLE tb1
SELECT
    xxx
FROM
    xxx
    WHERE
        xxx) t
distribute by rand();
```

### case 5
```
INSERT TABLE tb1
SELECT c1,c2
FROM tb2
WHERE xxx
sort by c1;
```
