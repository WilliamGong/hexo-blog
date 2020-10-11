---
title: Blog搭建实录-1 初始化hexo与建立git仓库
date: 2020-10-11T14:32:51+08:00
tags:
---
# 初始化Hexo文件夹
## 准备node.js
### Windows
在[这里](https://nodejs.org/en/)下载，然后一路安装。
### Linux
包管理工具，请。
## 安装Hexo
    npm install hexo-cli -g
## 建立Hexo文件夹
首先，在本地见一个文件夹，名字最好是英文,然后    

    npx hexo install <folder>
之后    

    cd <folder>
最后   

    npm install
一个Hexo文件夹就这样初始化好了。    
## 本地测试
> 其实是不需要的，我只是好奇而已。    
当然，之后这个操作是非常重要的，你可以把它当作熟悉Hexo操作。

首先，让Hexo生成静态网页

    npx hexo generate
然后，启动Hexo的本地服务

    npx hexo server
之后就可以访问localhost:4000访问本地的网页了。

# 建立Git仓库并上传到Github
> 如果你不知道什么是git, Github，你现在就可以把本BLog关了。
## 建立Github Pages发布仓库
在[Github](https://github.com)上，建立一个名为\<username>.github.io的仓库，然后放着。    
之后，Hexo文件夹用git初始化，commit之后push到Github。    
OK，其余的操作就主要在Github上进行了。