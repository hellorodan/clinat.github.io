---
layout:     post
title:      "GitHub package_lock.json 安全警报"
subtitle:   " \"GitHub Security Alerts\""
date:       2019-06-22 01:00:00
author:     "Clinat"
header-img: "img/post-bg-earth.jpg"
catalog: true
tags:
    - Problem
    - GitHub

---

> “GitHub安全警报：We found a potential security vulnerability in one of your dependencies. ”

## 问题描述

![](/img_post/github_security_alert.png)

GitHub提示：We found a potential security vulnerability in one of your dependencies.



## package.json和package_lock.json

NPM是随同NodeJS一起安装的包管理工具，在使用NPM进行相应的包进行安装的过程中，会再本地生成package.json文件和package_lock.json文件。

package.json文件的主要作用就是记录项目的所安装的依赖项，当node_modules文件夹被删除之后可以通过该文件中记录的依赖版本来重现安装依赖。

package_lock.json文件的主要作用是记录当前系统中所安装的各项依赖的版本，当执行`npm install package`

命令时，会在本地生成或更新该文件(只有NPM5之后才会生成)，同时也防止版本的自动更新。



## 解决方法

该问题的主要原因是项目依赖的版本和package_lock.json中版本(也就是当前安装的版本不匹配)。

### 安装Node.js

该部分不是本篇重点内容，而且安装简单，故不细讲。(以下基于Mac系统)

在[官网](http://nodejs.cn/download/)上下载MAC版本的.pkg文件，点击安装即可。

安装完成之后依次执行如下命令：

`curl http://npmjs.org/install.sh | sh`

`sudo npm update npm -g`

Npm.js其实是Node.js的管理工具，前面命令执行成功之后，执行该命令，如果成功输出版本号，说明npm安装成功：

`npm -v`

### 安装依赖

在安装依赖之前，先通过`git pull`命令同步本地和远端的数据。

然后调用`npm install package`命令来安装指定版本的依赖，该命令会将安装的版本信息更新到package_lock.json文件中。其中需要安装的依赖以及版本号可以通过 GitHub 的 Security Alerts 来查看。

最后通过 git 命令将更新之后的项目push回远端，问题可解。





