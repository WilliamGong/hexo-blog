---
title: icaurs使用过程中的坑
tags:
---
1. 配置文件需要generate一遍才会有，且在根目录下
1. widget的图标，fontawesome不可用,查证是内部cdn连不上，用其它的就行，比如这个

    https://cdn.bootcdn.net/ajax/libs/font-awesome/5.13.1/css/all.css
1. front-matter不要引用post_asset_folder目录的文件，原因应该是hexo在生成网页的时候会删除这个目录然后将文件夹内的文件移到父目录导致文件路径错误