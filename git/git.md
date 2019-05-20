# Git教程

## 创建版本库

```
git init 
```

## 克隆仓库

```
git clone 
```

## 工作区 暂存区 HEAD

```
git add     # 添加到暂存区

git commit      # 添加到 HEAD   

```

## 版本回退

```
git reset
```

## 撤销修改

1. 修改未提交到暂存区

```
git checkout -- <file>
```

2. 修改已提交到暂存区

```
git reset HEAD <file>
```

然后执行第一种方式。

## 删除文件

```
git rm <file>
```

## 远程仓库

```
git push
```