title: 搭建Blog拾遗及相关资料归档
date: 2018/05/30 15:23
tags:
 - hexo
 - blog
categories: 谈技论术
---

#### Node.js    
Ubuntu搭建Node.js环境，我只记两点：
+ 官网下载最新node版本  
[node官网](https://nodejs.org/en/)
+ 使用npm时出现：
```shell
Error: /usr/bin/env: node:没有那个文件或目录
```
> cause: ubuntu避免包冲突，会将nodejs操作命令改为nodejs，而不是node

  执行命令：
```shell
sudo ln -s /usr/bin/nodejs /usr/bin/node
```

#### Theme
##### 遇到的问题
本blog使用的主题为``material``，也许你可能在执行``hexo g``的时候可能会遇到如下错误：  
``Internal watch failed: watch ENOSPC``  
解决办法，执行命令：  
```shell
echo fs.inotify.max_user_watches=582222 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```
然后再执行``hexo g``.
##### 相关资料归档
+ [material-theme-github](https://github.com/viosey/hexo-theme-material)
+ [theme-doc](https://neko-dev.github.io/material-theme-docs/#/)
+ [icons](https://material.io/tools/icons/), 直接复制icon名称到主题``_config.yml``相应位置中
  ```yaml
  pages:
          关于我:
              link: "/aboutMe"
              icon: round-fingerprint-24px.svg
              divider: false
  ```
+ 图片压缩，小白玩法： [compress-image](http://www.yasuotu.com/)  
+ > 感谢教程：[GitHub+Hexo搭建个人Blog](https://my.oschina.net/ryaneLee/blog/638440)
