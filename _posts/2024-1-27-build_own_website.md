---
layout: post
title: "搭建自己的 github.io 个人网站"
date:   2025-9-25
tags: [web]
comments: true
author: yangchao
---

###### 说明：本教程只针对不了解网站搭建并且想要快速搭建起个人博客的新手，帮助建立网站的平台有很多，有一定网站开发基础的读者可另寻门路

<!-- more -->
- [Step 0. 准备工作](#step-0-准备工作)
- [Step 1. 建立博客仓库](#step-1-建立博客仓库)
- [Step 2. 修改仓库文件](#step-2-修改仓库文件)
- [Step 3. 开始写你的第一篇文章](#step-3-开始写你的第一篇文章)

## Step 0. 准备工作

利用 GitHub Pages 免费获取你自己的网站域名只需要一个先决条件：拥有你自己的 github 账号。如果你目前还没有，可以去 [GitHub 官网](https://github.com)先注册一个。

## Step 1. 建立博客仓库

借助 GitHub 平台搭建博客网站，首先要建立一个与你的 github 账号相关联的博客仓库。

仓库名称填写 `username.github.io`，注意 `username` 指的是用户名不是昵称。

## Step 2. 修改仓库文件

创建和修改文件来使页面与你的个人信息相关联。

- ### 创建 README.md文件

  README.md文件是对仓库的说明，一般来说第一行标题都是仓库名称，后面就是关于仓库的一些介绍。


- ### 修改 _config.yml 文件

  _config.yml 文件是博客网站的核心配置文件，里面包含了你想要显示在网站上的各种构造信息。

  - **网站名称和网站描述**

    ```c
    name: "xxx blog"
    description: "xxx的个人技术博客"
    ```

    这个根据你自己的喜好来设置。比如你可以给你的博客网站取一个好听点的名字，网站描述也可以是简短的自我介绍或个性签名等任何你想表达的内容。

  - **个人头像和网站 logo**

    avatar 代表头像，后面的链接是你想显示在页面的头像图片的 url。
    favicon 指网站图标，即显示在浏览器标签页和收藏夹里的 logo，通常以 32 * 32 像素大小的 .ico 图片为宜，也可以不设置。

  - **脚注和网址**

    ```
    footer-links:
    zhihu: 
    email: 576239368@qq.com
    facebook:
    github: https://yangchao315.github.io
    rss: # just type anything here for a working RSS icon
    twitter: 
    youtube:
  ```
  - **其他**

    如果你不知道改了之后会有什么后果，**不要去动它**。

- ### 删除 _posts 文件夹并重建

   _posts 文件夹里放的是博客文章.

- ### 删除 images 文件夹并重建

- ### 修改 about.md文件

   about.md里的内容是展示在“关于”页面上的。

## Step 3. 开始写你的第一篇文章

写文章之前需要熟悉如何在 github 上编写 Markdown 文件，这个网上有很多教程，可以自行参考。

文章文件的命名也是有讲究的，请按照下面的例子呈现的格式命名：

    2025-9-25-build_own_website.md

还有一点需要注意，每篇文章开头记得附上说明，格式如下：

    ---
    layout: post
    title: "文章标题"
    date:   2024-1-27
    tags: [tag1, tag2]
    comments: true
    author: xxx
    ---

- date 是写作日期，注意格式必须遵守 `YYYY-MM-DD` 才可以。

- tags 是文章标签，可以有一个或多个。

---

感谢阅读！
