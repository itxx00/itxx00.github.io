---
layout: post
title: "使用conventional-changelog和Strapdown.js为git仓库自动生成changelog html页面"
description: "通过简单的工具组合为git项目生成简单的changelog html"
categories: [shell]
tags: [changelog, conventional-changelog, strapdown.js]
---

> 一个项目的changelog对于使用者来说虽然不需要重点关注，但很重要

* Kramdown table of contents
{:toc .toc}

## 基本思路

通常软件产品对外发布时，我们需要提供一份changelog以告知使用者新版本所发生的变化，有两种方式可以产生需要的changelog内容， 一种是人工整理和编写，另外一种是通过工具实现自动化。这里介绍一种通过开源工具的组合快速实现自动生成的方法。


我们在开发过程中所有变更都会反映到git commit messages里面，git提交历史几乎可以反映软件的所有变更，基于此我们可以使用工具直接将git提交历史转化为changelog，再经过简单加工处理即可对外输出一个html页面。

## 规范提交

这就要求在代码提交过程中我们的commit message要规范化，其中一种被广为认可的规范名为约定式提交。详细可参考[约定式提交](https://www.conventionalcommits.org/zh-hans)
一个简单的提交类型参考如下：

- **build**: 变更仅影响工具出包或者build环境等外部依赖问题
- **ci**: 对CI配置的变更
- **docs**: 仅文档内容变更
- **feat**: 新特性
- **fix**: bug修复
- **perf**: 无bug修复/无新特性，仅性能提升
- **refactor**: 无bug修复/无新特性/无性能提升，仅重构
- **style**: 仅代码风格更改
- **test**: 仅测试代码变更

## 提交转化为markdown

有了规范的提交记录，下面就可以通过工具实现提交记录到markdown的转化。这里介绍一个工具叫conventional-changelog，命令行版本使用方法如下：
```
# install
npm install -g conventional-changelog-cli
# generate changelog markdown file
cd your-git-repo-project-home
conventional-changelog -p angular -i CHANGELOG.md -s -r 0
```
示例中用到的参数：
- -i : 读入已有changelog文件
- -p : 预设模板，可以是angular/atom/codemirror/ember/eslint/express/jquery/jscs/jshint
- -s : 写到目标文件名和-i指定的文件同名
- -r : 指定需要生成的release数量，0表示重新生成所有

更多参数可以执行```conventional-changelog --help```查看。

## markdown转化为html

这样我们就得到了一份名为CHANGELOG.md的历史变更记录文件，为markdown格式。接下来再通过另外一个工具名叫strapdown.js来自动生成html。

strapdown.js是一个js文件，不需要像上面生成markdown那样在server端生成，只需要在单个html页面中引入该js文件即可实现从markdown自动渲染出html页面。详细可参考[strapdown.js](https://strapdownjs.com/)

使用方法如下：
```
cat >changelog.html <<"EOF"
<!DOCTYPE html>
<html>
<title>XXX Changelog</title>
<meta charset="utf-8">
<xmp theme="darkly" style="display:none;">
EOF

cat CHANGELOG.md >>changelog.html
cat >>changelog.html <<"EOF"
</xmp>
<script src="http://strapdownjs.com/v/0.2/strapdown.js"></script>
</html>
```
这样我们就通过拼接的方式生成了一份changelog.html。需要注意的是changlog内容中不能包含```</xmp>```关键字。

over.
