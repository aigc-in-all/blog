---
layout: post
title: "在多台电脑上写Octopress博客"
date: 2014-01-18 16:22:18 +0800
comments: true
categories: Octopress
---
我将介绍如何在多台电脑上部署Octopress博客，我假设你已经在一台电脑上面部署过Octopress并且已经push到GitHub上面。如果你还不会或者不知道Octopress是什么的话，建议你先看看[Octopress](http://octopress.org/)的官网，然后跟着[Document](http://octopress.org/)一步步部署自己的博客。如果你弄坏了本地的Octopress配置，或者是你重装了系统，又或者是你在另一个电脑上面想继续编辑Octopress博客的话，那么这篇文章会告诉你该怎么做。

<!-- more -->

### Octopress原理 ###
这里以部署到GitHub为例，Octopress的git仓库(repository)有两个分支，分别是`master`和`source`。`master`存储的是生成的博客网站本身，而`source`则存储的是生成博客的源文件。`master`所存储的内容可以通过`source`生成，也就是说，只要有源代码(`source`分支里面所存储的内容)就可以部署博客。`master`的内容在根目录的`_deploy`文件夹内，当你push源文件时会忽略，它使用的是`rake deploy`命令来更新到`master`分支上面去的。

### 创建一个本地的Octopress仓库 ###
重新创建一个已经存在的本地Octopress仓库只需要以下几个步骤：

#### 克隆(clone)博客内容到本地 ####
首先将博客的源文件(`source`分支)clone到本地的octopress目录：

```
$ git clone -b source git@github.com:username/username.github.io.git octopress
```

然后将博客网站内容(`master`分支)clone到`octopress`文件夹下的`_deploy`目录里：

```
$ cd octopress/
$ git clone git@github.com:username/username.github.io.git _deploy
```
然后安装依赖（如果之前在本机部署过octopress博客的话，这一步可以略过）：

```
$ gem install bundler
$ rbenv rehash    # If you use rbenv, rehash to be able to run the bundle command
$ bundle install
```
生成博客

```
$ rake generate
```
这样就建好了一个新的本地仓库了。

#### 更新和推送 ####
当你要在另一台电脑上写博客或者修改博客配置的话，首先更新`source`支分。更新`master`不是必须的，因为你更改源文件后还是需要`rake generate`的，它会更新本地的`_deploy`文件夹下的内容。

```
$ cd octopress
$ git pull origin source  # update the local source branch
$ cd ./_deploy
$ git pull origin master  # update the local master branch
```

写完博客之后不要忘了推送到remote，下面的步骤在每次更改之后都必须做一遍。

```
$ rake generate
$ git add .
$ git commit -am "Some comment here." 
$ git push origin source  # update the remote source branch 
$ rake deploy             # update the remote master branch
```