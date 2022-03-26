---
title: Gentoo 中安装 GNU Octave 的注意事项
date: 2022-03-26T22:31:18+08:00
toc: true
cover:
thumbnail:
categories:
    - 杂谈
tags:
    - 杂谈
    - Gentoo
    - Octave
---
在 Gentoo 安装 octave 时，如果只使用官方默认的 USE，你会惊喜的发现 pkg 是无法下载软件包的：
```
support for URL transfers was disabled when Octave was built
```
这是因为 Gentoo 在对 ocatve 配置的 USE 中，没有加上对 curl 的支持，导致 pkg 无法使用 URL 下载软件包。    
如果需要使用 pkg 安装软件包，需要为 octave 单独加上 USE`curl`。    
在`/etc/portage/package.use`中，加上：

    sci-mathematics/octave curl
即可。

PS: 记得`emerge -avuDN @world`