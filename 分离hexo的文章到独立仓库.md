---
title: 分离 hexo 的 _posts 目录到独立仓库
date: 2022-07-31 00:49:03
categories: 
 - hexo
tags: 
---

距离上一次搞 hexo 应该过去好几年了，最近突然又起了兴趣。今天便对着 hexo 的文档捣鼓了一下午，终于成功在 Github Pages 上部署了新的博客页面。在捣鼓的过程中也发现了一个不太完美的地方，那就是文章的源码是嵌入在 hexo 的源码里的，这一点让我非常难受啊。

后续捣鼓博客主题的时候，发现主题的源码都是通过 git 子模块添加到 hexo 源码内的，这时突然想到存放文章的目录 `source/_posts` 是不是也能给他搞成引用外部仓库的子模块呢？这样我就可以单独去维护和浏览文章的源码。如果哪天不想再用 hexo 了，直接把存 hexo 源码的仓库删了就完事了，反正我都文章都在另一个独立的仓库里。

简单验证了一下设想，确实是可行的，本文就记录一下 `source/_posts` 分离到一个独立仓库的方法。

## 使用 hexo 搭建博客
hexo 搭建博客和部署到 Github Pages 的过程看官方文档就行了，写得挺详细的，也有中文，直接看文档吧。   
https://hexo.io/zh-cn/docs/

hexo 生成的静态页面我也是发布到一个独立的仓库里，命令一键发布的方式看这。    
https://github.com/hexojs/hexo-deployer-git 

这部分内容百度谷歌都能查到很多资料，也不是本文的重点，我就不写了。

## 分离 source/_posts 目录
hexo 的使用流程走完后，应该是能看到有个 `source/_posts` 目录的，这个目录用来存放文章的源码，我想做的就是把他配置成 git 子模块，指向一个独立的文章存储仓库。    

第一步肯定就是先建一个文章仓库，至于仓库放在哪都无所谓，github 也行，码云也行，是不是开源的也都无所谓，爱咋咋滴。   
我是在 github 上建了个叫 "blog-posts" 的新仓库，仓库准备好了就可以进入下一步了。

第二步是进到 hexo 的源码目录，把 `source/_posts` 删掉，然后配置一下 git 子模块，把刚刚创建的 "blog-posts" 仓库添加进去，命令如下。
```bash
git submodule add git@github.com:zjlian/blog-posts.git source/_posts
```
命令中间的仓库链接部分 `git@github.com:zjlian/blog-posts.git` 自行替换成自己的博客仓库。

最后一步就是将修改后的 hexo 源码仓库提交到云端上去。   

很简单，到这里就完工了。   

## 分离后怎么写文章
这部分比较灵活，想怎么用就怎么用，这里简单列两种用法。   
（一）单独 clone 文章仓库下来写，这种用法需要自己手动编写 hexo 博客的 [Front-matter](https://hexo.io/zh-cn/docs/front-matter) 信息，文章写完了，先提交文章仓库，然后到 hexo 仓库内用命令更新同步子模块，把最新的文章都下载下来。
```bash
git submodule init
git submodule update --remote --merge
``` 
更新完后再敲一下部署命令 `hexo deploy` 就可以了。   

（二）第二种用法是直接在 hexo 仓库里写，用 hexo 的新建文章命令 `hexo new '标题'` 创建文章，写完后直接就能用命令 `hexo deploy` 部署，然后再进到 `source/_posts` 目录里把文章提交到云端仓库上。

用法没有什么固定的规则，可以根据自己的习惯和需求灵活变通。   
有了习惯用法后可以写个脚本，自动完成重复工作。   
也可以买台云服务器，捣鼓个 DevOps 服务出来，配合上 github 的 webhook，文章仓库提交更新后，让 github 自动给  DevOps 服务发请求，服务器收到请求后自动下载更新 hexo 仓库和文章仓库，再自动执行部署命令。

(●'◡'●)
