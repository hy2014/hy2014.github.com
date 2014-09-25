title: git rebase
date: 2014-07-25 17:06:46
tags: git
category: git
description: "git rebase 笔记"
---

今年开始，项目要从SVN转到了git, 写一篇博客总结一下git rebase的功能。

<!-- more -->

##描述
rebase从我的理解就是， 讲别的修改重新在你指定的branch上执行一遍， 如果读者还是感觉不太清楚, 我将通过一些应用场景去解释。

##场景1 - 多人协同开发

我们通常会遇到这种情况，一个项目，多个人同时开发，当他们修改同一个文件的时候，好点的情况，git会进行auto merge。糟糕的话，会发生文件冲突文件冲突。 对于冲突，我们无法避免。 先讲讲auto merge的缺点。

auto merge有一个缺点是， 会造成你当前branch的分支历史混乱，不利于Scrum Master查看commit的执行history.

使用rebase可以避免这个情况，这种用法其实很简单，就是当你在pull的时候， 只是需要简单加上-r这个参数。

	git pull -r origin master

时间久了， 这个branch的commit history就是一条笔直的线， 不会产生分支， 所以我认为再pull的时候加上-r应该当作一个好习惯。

##场景2 - branch间进行rebase

在项目的持续开发中,  我们会有多个branch， 假定现在有两个branch， production和develop。 

	production - 线上正在运行的版本
	develop - 正处于新功能开发的版本

这个时候如果production上有一个hotfix,那么我们将要做这么几个事情。

	a. 在production branch上做hot fix。
	b. 将production上的这个hot fix rebase到develop branch上。

在这里, 将定已经在master上提交了hotfix, 首先看一下该hotfix的commit

	git log --oneline -3

找到这个commit, 假定是`d413a16`, 接下来使用rebase命令
	
	git rebase develop d413a16 -i

`-i`这个参数表示交互,执行此命令后, 会进入到编辑文件的环境, 文件内容显示的是这次会rebase的哪些commit(git 默认是会把两个branch上次相同commit之后的commit作为起点, 终点是`d413a16`这个commit)。

如果我们不需要将所有的commit rebase到develop, 只是rebase `d413a16`, 使用'#'将别的commit注释掉, 然后`:wq`, 退出。 git便自动开始rebase(这一步可能会发生冲突, 冲突解决后文再介绍)。

rebase完成之后,你会发现你进入到了一个游离态的branch, 到底发生了什么？

	其实,rebase并没有将当前d413a16 rebase到develop这个branch, 而是rebase到了一个新的，游离的branch, 该游离branch的commit历史是develop + d413a16。

所以我们需要将develop的HEAD指向这个游离态就可以了, 我们先切换到develop这个branch。
	
	git checkout develop

然后查看HEAD的log
	
	git reflog --oneline -10

找到游离态的HEAD, 然后改变当前branch的HEAD。
	
	git reset HEAD@{X} ---- X是游离态的HEAD标识, 是一个整数, 从文本操作顺序， 应该是1

这个时候,你查看一下, 会发现develop的代码没有变化, 这又是为什么呢？

	使用git status查看一下, 你的代码有一些modified。 其实d413a16这个commit已经apply到了这个develop branch, 但是你本地develop的代码内容d413a16之前的内容, 而modified的文件是当前的内容相对d413a16 commit而言做的修改.

所以只需要回滚到d413a16就可以了

	git checkout .

到此为止, 完成目标。

##rebase - 解决冲突
如果发现冲突，git会指示你使用merge tools手动resolve冲突。在这个过程中可以使用下面的命令

	git rebase --continue

解决完冲突，就可以像往常一样，正常提交了。