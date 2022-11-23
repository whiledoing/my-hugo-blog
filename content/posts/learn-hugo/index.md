---
title: "learn hugo blog generation framework"
subtitle: ""
date: 2022-11-23T18:10:25+08:00
lastmod: 2022-11-23T18:10:25+08:00
draft: true
author: "whiledoing"
tags: [hugo, golang]
categories: [software]

resources:
- name: featured-image
  src: images/featured-image.jpeg
- name: featured-image-preview
  src: images/featured-image.jpeg
---

记录折腾博客生成引擎[hugo](https://gohugo.io/getting-started/quick-start/)学习内容

<!--more-->

## hugo man

hugo server自带livereload功能（原理是hook js代码到返回的html页面，进而在加载时，和服务端构建ws连接执行refresh推送）。

通过`hugo server --navigateToChanged`命令，可以自动加载到最新变更的文件

## themes

### LoveIt

国人开发，支持贴近国人生产相关的shortcodes，使用文档可以[参考](https://hugoloveit.com/zh-cn/)