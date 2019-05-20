# Git教程

在使用 git 之前，建议使用别名设置 `git log`，这样看日志会更舒服。设置完成后输入`git lg`即可显示操作记录。命令如下：

`git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"`

## 版本管理

1. 创建版本库
```
git init 
```
2. 工作区 暂存区 HEAD
```
git add     # 添加到暂存区
git commit      # 添加到 HEAD   
```
3. 版本回退
如果需要查看历史版本记录，可以使用 `git reflog`。
```
git reset --hard <hash_num>
```
4. 撤销修改
修改未提交到暂存区
```
git checkout -- <file>
```
修改已提交到暂存区
```
git reset HEAD <file>
```
然后执行 `git checkout`。
5. 删除文件
```
git rm <file>
```

## 远程仓库

1. 关联远程仓库
需要先创建远程仓库。
```
git remote add origin <远程仓库地址>
```
2. 第一次推送
```
git push -u origin master
```
第一次推送master分支时，加上了`-u` 参数，Git 不但会把本地的 master分支 内容推送的远程新的 master分支，还会把本地的 master分支 和远程的 master分支关联起来，在以后的推送或者拉取时就可以简化命令。
3. 克隆远程库
```
git clone <远程库地址>
```

## 分支管理

1. 查看分支：git branch
2. 创建分支：git branch <name>
3. 切换分支：git checkout <name>
4. 创建+切换分支：git checkout -b <name>
5. 合并某分支到当前分支：git merge <name>  # 合并分支时，加上`--no-ff`参数就可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并，而 `fast forward`(默认使用该模式)合并就看不出来曾经做过合并。
6. 删除分支：git branch -d <name>  # 如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name> 强行删除。
7. bug分支：git stash 一下，然后去修复bug，修复后，再git stash pop，回到工作现场。

## 多人协作
查看远程库信息，使用git remote -v

多人协作的工作模式通常是这样：
1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！
5. 如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。

## 标签管理

1. `git tag <tagname>`: 用于新建一个标签，默认为HEAD，也可以指定一个commit id；
2. `git tag -a <tagname> -m "blablabla..."`: 可以指定标签信息；
3. `git tag`: 可以查看所有标签。
4. `git push origin <tagname>`可以推送一个本地标签；
5. `git push origin --tags`: 可以推送全部未推送过的本地标签；
6. `git tag -d <tagname>`: 可以删除一个本地标签；
7. `git push origin :refs/tags/<tagname>`: 可以删除一个远程标签。

## 疑难杂症

