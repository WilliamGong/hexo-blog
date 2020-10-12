---
title: Blog搭建实录-2 配置Github Actions
date: 2020-10-11T15:06:57+08:00
toc: true
categories:
- 博客搭建
tags: 
- 博客
---
# Github Actions介绍
> 注：以下内容大部分参考了[这篇post](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)，这篇post讲的比我清楚多了，人家是专业的。
## 什么是Github Actions?
Github Actions是Github自己推出的[持续集成服务](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)，可以自动地进行各种各样的构建并发布到正确的地方。     
在本Blog中，我就使用了Github Actions来自动构建Hexo的静态网页并将它发布到Github Pages上。    
这些构建，发布之类的操作，在Github Actions中被称为actions。用户可以将actions写成独立的脚本并供给其他人使用。Github建立了一个官方市场，可以找到我们需要的actions。
## 术语介绍
+ workflow（工作流）：指运行一次的所有流程；
+ job（任务）：组成workflow的单元，一个workflow由多个job组成；
+ step（步骤）：job执行时执行的单元，由多个action组成；
+ action：这就不多说了吧。

# 实战
## 获取 Personal Access Token
打开你的Github 账户设置页，转到Developer settings -> Personal access tokens，生成时记得勾选repo项，admin:repo_hook和workflow项。    
之后复制生成的字符串，回到hexo仓库，打开仓库设置，转到Secrets，把字符串以环境变量的形式存储。变量名凭喜好自取。
## 配置Actions
首先我们在GitHub打开Hexo的仓库，转到actions选项。      
根据网页的提示，建立一个workflow。这样你就会进入一个编辑.yml文件的界面，文件就是workflow的配置文件。    
这时在右边有市场界面，让我们在里面搜"hexo"，可以看到许多发布hexo博客的actions，这里我选择的是[hexo-deploy](https://github.com/Solybum/hexo-deploy),选择版本，将代码框的内容粘贴到workflow文件中，按注释改一下配置，保存。
### 注意
根据actions的不同，所需要的token/key类型也不同。有的使用Personal Access Token(PAT)，有的使用ssh key，具体看action的说明。    
我倾向于使用PAT，因为PAT只用存储在hexo仓库上。相比之下，用ssh key需要将公钥放在hexo仓库，私钥放在pages仓库，较为麻烦。
# 最后
将hexo仓库push一下，actions就会自动运作，几分钟后Blog就可以访问了。