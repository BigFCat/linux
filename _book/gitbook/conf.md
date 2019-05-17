# Gitbook 配置

## book.json

```json
{
    "title": "牟梦云的Linux学习笔记",
    "author": "牟梦云",

    "links" : {
        "sidebar" : {
            "Home" : "https://www.24linux.com"
        }
    },

    "plugins": [ "toggle-chapters", "github", "github-buttons", "-lunr", "-search", "search-plus", "gtoc" ],

    "pluginsConfig": {
        "theme-default": {
            "showLevel": true
        },

        "github": {
            "url": "https://github.com/mumengyun"
        },

        "github-buttons": {
            "repo": "mumengyun/linux",
            "types": [
                "star",
                "watch",
                "fork"
            ],
            "size": "small"
        },

        "terminal": {
            "copyButtons": true,
            "fade": false,
            "style": "flat"
        }
    }
}
```

## 插件说明

1. toggle-chapters：折叠左边树形目录，安装后只会显示目前使用章节的子章节
2. github：右上角 github 链接
3. github-buttons： 项目star
4. search-plus： 支持中文搜索

## 插件安装

在 `book.json`.`plugins` 添加需要安装的插件名称，执行 `gitbook install`。 有些插件需要配置 `pluginsConfig` 才会生效，有的可以直接使用默认配置。

## 目录分层

整个 gitbook 的目录结构主要由 `SUMMARY.md` 文件配置。

```
# Summary

* [编写说明](README.md)

## 常用工具
* [Git教程](git/git.md)
    * [Git常用命令](git/git-use.md)
    * [Git安装配置](git/git-install.md)
* [Gitbook教程](gitbook/gitbook.md)
    * [Gitbook配置](gitbook/conf.md)
    * [Gitbook & Github](gitbook/github.md)
```

`##` 用于分块。层级结构以目录的形式构成。

## Gitbook 命令

`gitbook buid` : 创建发布文件 

`gitbook init` : 根据 SUMMARY.md 文件配置，创建目录结构

`gitbook serve` : 本地快速预览

`gitbook install` : 根据 book.json 文件配置，安装插件

---

参考

[gitbook 中文文档](https://chrisniael.gitbooks.io/gitbook-documentation/content/index.html)