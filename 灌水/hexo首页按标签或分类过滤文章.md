---
title: hexo 首页按标签或分类过滤文章
date: 2022-07-31 18:46:09
categories:
 - hexo
tags:
 - 灌水
---

有时候写了一些很水的文章，不太希望它出现在首页时，就需要这样的过滤功能。    

原本是想自己写的，简单修改了首页的 ejs 模板后，发现效果并不符合需求，按照 `_config.yml` 内的配置项，首页展示设置了分页，每页 10 篇文章，如果这里 10 篇都带有需要过滤的标签，最后会得到以一个空白的页面，特别蠢。

想要正确过滤，并且保证每页的文章数量都是设定数量，就必须从 hexo 的实现源码上下手。在我搜索 hexo 的拓展开发文档时，无意间发现我的需求早就有人实现了。

这个 hexo 拓展的名字叫 "hexo-generator-index2"。   
github 链接：https://github.com/Jamling/hexo-generator-index2

用法也很简单，安装命令如下：
```bash
npm install hexo-generator-index2 --save
```

安装后只需要在 `_config.yml` 文件内添加如下内容即可：
```yaml
# index2 generator 配置首页文章过滤
index2_generator:
  exclude:
    - tag 灌水
```
添加了上面的配置后，生成的首页就能过滤掉带有 "灌水" 标签的文章。

除了过滤还可以设置白名单，只有特定标签或分类的文章才出现在首页。更多的信息可以到 github 仓库上看。

(●'◡'●)
