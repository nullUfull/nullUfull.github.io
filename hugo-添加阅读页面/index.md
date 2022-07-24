# Hugo-添加阅读页面

创建博客的起初，就想有一个页面用来置放看过的书籍、电影等，之前用hexo搭建博客，其实就有个豆瓣的插件，后来迁移到了hugo也一直在找，但是可惜没有找到。
本周一时兴起，想再找找，就看到了一个豆瓣书影音同步的GitHub Action：[doumark-action](https://github.com/lizheming/doumark-action) ，这个 Action可以帮我们拉到指定用户的书籍、影视信息并保存下来，这样一来有了数据，就可以通过网页将其渲染出来就好。

## 配置GitHub Action
配置Github Action，在博客仓库新建`.github/workflows/douban.yml`文件，配置正确用于拉取数据

```
# .github/workflows/douban.yml
name: douban
on: 
  schedule:
  - cron: "0 5,11,18,23 * * *" // 支持cron配置
  watch:
    types: [started] // 仓库收到star会自动触发workflow

jobs:
  douban:
    name: Douban mark data sync
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: book
      uses: lizheming/doumark-action@master
      with:
        id: 161758056
        type: book
        format: csv
        dir: ./data/douban

    # - name: movie
    #   uses: lizheming/doumark-action@master
    #   with:
    #     id: 161758056
    #     type: movie
    #     format: csv
    #     dir: ./data/douban
  
    - name: Commit
      uses: EndBug/add-and-commit@v8
      with:
        message: 'chore: update douban data'
        add: './data/douban'
```

## 添加阅读页面
1. 添加阅读页面，新建文件，`content/books/index.md`

```
---
title: "阅读"
type: static
layout: books
---
```

2. 添加页面渲染文件，`layouts/static/books.html`，可参考我的阅读页面html文件：[books.html](https://github.com/nullUfull/nullUfull.github.io/blob/master/layouts/static/books.html)

3. 菜单栏添加阅读入口
```
...
 [languages.zh-cn.menu]
      [[languages.zh-cn.menu.main]]
        weight = -1
        identifier = ""
        pre = ""
        post = ""
        name = "主页"
        url = "/"
        title = ""
      [[languages.zh-cn.menu.main]]
        weight = 1
        identifier = ""
        pre = ""
        post = ""
        name = "阅读"
        url = "/books"
        title = ""
...
```
## 踩坑
配置好GitHub Action后，触发这个Action执行后，发现一直跑不过，一度怀疑是不是配置错了以及我的豆瓣账号是否有问题，改了好几次配置然后触发执行，甚至还在别人blog帖下留言帮忙看看为啥一直跑不过。

之前是想配置个拉取电影信息的试试，没想到一直跑不过，后面改成拉取书籍信息试试，结果成功了，应该是作者提供的拉取豆瓣电影数据接口存在的问题，而不是Action配置问题。

## 参考
[豆瓣书影音同步 GitHub Action](https://imnerd.org/doumark.html)

[「电影 / 阅读」页面，顶！](https://immmmm.com/doumark-action/)
