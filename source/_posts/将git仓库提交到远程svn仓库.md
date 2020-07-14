---
title: 将git仓库提交到远程svn仓库
date: 2020-07-14 10:35:12
tags:[git,svn]
---

公司代码日常开发由于SVN过于缓慢，已经迁移到内部gitlib服务器上，并使用git管理。但由于研发管理仍采用SVN，导致开发完成提测和发布仍需要将代码上传SVN。由此需要将本地已有GIT仓库上传到SVN服务器。

##创建SVN仓库
```
svn co http://svn.example.com/foo
cd myproj
svn mkdir trunk
svn commit -m'Created trunk directory'
```
或者直接在现有仓库创建目录
```
svn mkdir --parents http://url/dir_name --message "messages"
```

##手动指定SVN仓库

在本地git仓库配置文件中添加以下内容：
```
vim .git/config

[svn-remote "svn"]
        url = http://svn.example.com/foo/trunk  
        fetch = :refs/remotes/git-svn
```
可以使用git配置检查指令来验证手动修改的配置：
```
git config --local -l
svn-remote.svn.url=svn://guan.isum.cn/smart/shells
svn-remote.svn.fetch=:refs/remotes/git-svn
```
此配置作用为关联了SVN仓库和本地git仓库

##提交本地git仓库到SVN

###拉取远程SVN仓库,并checkout到本地git仓库的git-svn分支上

```
git svn fetch svn
git checkout -b svn git-svn
```
###将master分支merge进svn分支并提交到svn库
```
git merge master --allow-unrelated-histories
git svn dcommit
```

###rebase到svn分支以便从master版本推送到svn库
```
git checkout master
git rebase svn
git branch -d svn
```