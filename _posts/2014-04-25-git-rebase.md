---
layout: post
title: "git rebase"
description: ""
category: git
tags: [git]
---
{% include JB/setup %}

今年， 项目将要从SVN转到了git， 关于git的使用， 我还是用svn的思想去使用git。 不过最近， 学会了一个rebase的命令。
rebase分为两种， 本文主要介绍的是协同开发处理冲突的rebase。

	一般，我们通常会遇到这种情况， 一个项目， 两个人同时开发。 当提交的时候， 好点的话， 只是auto merge一下。 如果糟糕点， 就直接显示有些文件冲突。 对于冲突， 我们稍后再说， 先讲讲auto merge的缺点。

auto merge会在你的提交记录中显示， 也就是说， 如果你commit了一次， 但是因为pull的时候， auto merge。 那么它会把auto merge也当作一次commit， 这是第一个缺点。
第二个缺点是, auto merge造成的分支历史非常不清晰。

那么该怎么使用rebase呢， 简单介绍一下。 当你在pull的时候， 只是需要简单加上-r这个参数

	git pull -r origin master

如果，发现冲突了， 使用merge tools手动resolve冲突之后， 可以使用下面简单的命令

	git rebase --continue

最后就可以像往常一样，正常提交了。


