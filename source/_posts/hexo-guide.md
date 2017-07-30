---
title: 使用Hexo+GitHub搭建及配置个人博客
date: 2017-07-31 01:09:13
tags: [Hexo, GitHub]
---

## 搭建前的步骤
1. 在GitHub上创建好Repository，name必须以`username.github.io`的形式来命名，否则不能成功部署
2. 提前安装好[Git](https://git-scm.com/)和[Node.js](https://nodejs.org/en/)（已安装的略过）
3. 安装[Hexo](https://hexo.io/)：安装Node.js后即可通过npm来安装Hexo：
``` bash
$ npm install hexo-cli -g
```

## 初始化项目
执行以下命名创建并初始化项目，会自动帮我们下载/生成好相关模板文件，关于各文件及文件夹的意义，可参考[官方文档](https://hexo.io/zh-cn/docs/setup.html)：
``` bash
$ hexo init <folder>
$ cd <folder>
$ npm install
```
初始化完后，先安装`hexo-deployer-git`插件，因为后续我们将把代码上传至GitHub：
``` bash
$ npm install hexo-deployer-git --save
```

## 站点基本配置
在主目录下的`_config.yml`中，配置以下基础信息，其中`language`修改为`zh-Hans`即可改成简体中文，对应主题文件夹中的`languages`文件夹下的`zh-Hans.yml`，自己看着来改就好了，有些主题是`zh-CN.yml`
```
# Site
title: codezjx's Home
subtitle: 
description: 
author: codezjx
language: zh-Hans
timezone:
```

接下来配置GitHub设置，类型设置为git，指定好repo地址，branch必须设置为master，因为GitHub Page只会从mater分支生成。（**注意有坑**：这里我们需要单独设置好在GitHub上使用name和email，否则将会使用global的user.name和user.email，囧~~~）：
```
deploy:
  type: git
  repo: https://github.com/codezjx/codezjx.github.io.git
  name: codezjx
  email: code.zjx@gmail.com
  branch: master
```
题外话，这里简单说一下`hexo-deployer-git`插件的工作流程：当执行部署操作的时候，首先会自动初始化git仓库（位置在.deploy_git中），并关联到指定repo与branch，后续public文件夹中自动生成的页面代码将会拷贝至此目录中进行代码管理。若修改了name和email，需要删掉整个.deploy_git再重新部署才会生效。有兴趣的可以看下[hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)的源码。


## 主题配置
首先当然是下载主题了，题主用得是Hexo上比较热门的[Next](https://github.com/iissnan/hexo-theme-next)主题，简洁大方好看嘿嘿，clone到themes目录中即可：
``` bash
$ cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
修改主目录下的`_config.yml`，指向我们刚刚clone的主题：
```
theme: next
```
在对应主题目录下，同样有个`_config.yml`文件，可以对主题进行一些个性化的定制。例如在Next主题中，我们可以设置社交账号及其对应的icons。若对Next主题有兴趣的，可以参考官方的[配置文档](http://theme-next.iissnan.com/getting-started.html)：
```
social:
  GitHub: https://github.com/codezjx
  Twitter: https://twitter.com/codezjx
  StackOverflow: https://stackoverflow.com/users/3919425
  Weibo: http://weibo.com/2350975412

social_icons:
  enable: true
  GitHub: github
  Twitter: twitter
  StackOverflow: stack-overflow
  Weibo: weibo
```

## 部署到GitHub上
主要用的是三部曲：

 - `hexo g`: 生成静态文件，在public文件夹中（hexo generate的缩写）
 - `hexo s`: 生成本地预览，默认情况下，访问网址为： http://localhost:4000/
 - `hexo d`: 部署并提交代码至GitHub中（hexo deploy缩写）
 
还有一个比较重要的命令：
 
 - `hexo clean`: 清除缓存文件 (db.json) 和已生成的静态文件 (public)。在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。

最后，打开我们创建好的GitHub Page页面：`username.github.io`，这个时候应该能看到Next主题默认的样式了，此时博客系统已基本搭好了，Have fun~