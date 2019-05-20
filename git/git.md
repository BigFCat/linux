# Git教程

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

```
git reset --hard <hash_num>
```

如果需要查看历史版本记录，可以使用 `git reflog`。

4.  撤销修改

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
5. 合并某分支到当前分支：git merge <name>
6. 删除分支：git branch -d <name>

## 测试

