---
title: git 常用命令
date: 2018-05-25 21:14:37
tags: git
---
## git常用操作
### 公钥
```
ssh-keygen -t rsa
```
一般公钥存在`~/.ssh/id_rsa.pub`，并且需要你上传到github，gitlab等。

<!-- more -->
### 分支相关
- `git branch`：查看本地有那些分支，结果branch前面带*号的是当前分支。
- `git branch -a`：查看一共有哪些分支：,会列出所有分支，你可以根据分支名拉取分支代码
- `git checkout -b $branch_name`：如果分支不存在，新建一个名字是`$branch_name`的分支，如果分支存在，拉取分支代码，并切换到分支：
- 上面的代码也相当于以下两行命令：  
    `git branch $branch_name`  
    `git checkout $branch_name`   
- 切换分支：`git checkout $branch_name`
- 改完了代码，提交：`git commit -m '$text'`  
- 提交分支：`git push origin [name]`,如果不加`name`，会提交所有分支。
- 删除分支：`git branch -d [name]`
- 合并分支：`git merge [name]`，把分支合并到当前分支
- 拉取分支的更新：`git checkout master;git pull origin`,拉取master的更新
- 查看已经合并了的分支：`git branch --merged`,这个列表里面不带`*`的都可以删掉了，因为已经merged
- 查看未合并的分支：`git branch --no-merged`,这个不能删除，只能用`git branch -D`强制删除
