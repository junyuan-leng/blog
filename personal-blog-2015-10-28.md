Title: 批量修改 Git 历史提交信息
Date: 2015-10-28
Category: 折腾

之前为了保存个人博客的文章，在 GitHub 开了个 repo，每次更新文章后就 push 一下，结果发现有若干次 push 之后 GitHub 上并没有对应的contribution，用 git log 查了一下，才发现由于 VPS 上没有用 git config 配置 name 和 email，造成 git 自动将用户识别成了 VPS 的 username@hostname 的形式

于是首先赶紧 git config 配置下

	git config user.email "junyuanleng@gmail.com"
	git config user.name "deepurple"

如果用 --global 选项的话，是全局设定，不用 --global 的话，只对当前所在的 repo 生效，设置完确认下即可

	git config --list

但是配置后的用户信息只对之后的 commit 生效，之前的历史提交批量修改的话，就需要参照 GitHub 官方给出的[指南](https://help.github.com/articles/changing-author-info/)，具体步骤翻译如下

#改变作者信息

为改变已经存在的 commit 的用户名和/或邮箱地址，你必须重写你 Git repo 的整个历史。

>警告：这种行为对你的 repo 的历史具有破坏性。如果你的 repo 是与他人协同工作的，重写已发布的历史是一种不好的习惯。仅限紧急情况执行该操作。

###使用脚本改变你 repo 的 Git 历史

我们写了一段能把 commit 作者旧的邮箱地址修改为正确用户名和邮箱的脚本。

>注意：执行这段脚本会重写 repo 所有协作者的历史。完成以下操作后，任何 fork 或 clone 的人必须获取重写后的历史并把所有本地修改 rebase 入重写后的历史中。

在执行这段脚本前，你需要准备的信息：

* 欲修改的旧的邮箱地址
* 正确的用户名和邮箱地址

1.打开终端（Mac 或 Linux 用户）或命令行（Windows 用户）

2.创建一个你的 repo 的全新裸 clone （repo.git 替换为你的项目，下同）

	git clone --bare https://github.com/user/repo.git
	cd repo.git

3.复制粘贴脚本，并根据你的信息修改以下变量：
	
	OLD_EMAIL
	CORRECT_NAME
	CORRECT_EMAIL

脚本：

	#!/bin/sh

	git filter-branch --env-filter '

	OLD_EMAIL="your-old-email@example.com"
	CORRECT_NAME="Your Correct Name"
	CORRECT_EMAIL="your-correct-email@example.com"

	if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
	then
	export GIT_COMMITTER_NAME="$CORRECT_NAME"
	export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
	fi
	if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
	then
	export GIT_AUTHOR_NAME="$CORRECT_NAME"
	export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
	fi
	' --tag-name-filter cat -- --branches --tags

4.按 Enter 执行脚本

5.查看新 Git 历史有没有错误

6.把正确历史 push 到 Github

	git push --force --tags origin 'refs/heads/*'

7.清除临时 clone

	cd ..
	rm -rf repo.git

搞定