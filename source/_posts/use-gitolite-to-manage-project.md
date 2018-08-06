---
title: 使用gitolite搭建Git服务器
date: 2018-07-27 20:23:20
tags: 
- Git相关
---

# 前言 #

2018年4月底入职一家研发智能机顶盒的公司，7月中旬一位大神离职了，然后进行工作交接，其中就涉及到了如何利用Gitolite管理项目。Git操作使用的很多，但是如何搭建Git服务器以及如何进行管理却还没有接触过，因此利用闲暇时间摸索了下并记录下来~~
我个人买了个阿里云的服务器，一年是293￥，其实也不算贵了。 主要原因是：
1. 这毕竟是自己个人学习的内容，当然不能直接在公司服务器进行测试了；
2. 以前将个人博客托管在github或者gitlab，总是会丢数据，所以打算将博客搭建到个人服务器，方便管理；
3. 后期如果精力的话会实现一些个人项目，可以放在自己的服务器中。

<!-- more --> 

# 正题 #

言归正传，我们现在开始着手搭建Git服务器，主要流程有新增用户->安装软件->配置SSH KEY->管理员配置->新增用户和项目。

## 新增服务器用户 ##

**初始环境**

|名称|详情|
|-|:-:|-:|
|服务器|阿里云|
|配置|1核 1G内存 40G|
|操作系统|Ubuntu 16.04|

默认情况下服务器只有root用户，直接用root进行操作肯定是不安全的，因此需要额外新增两个账户。一个用来存放Git仓库（用户名为git），一个用来进行Git用户管理（用户名为git_admin）。


	useradd git -m //添加git用户
	passwd git//设置git用户的登陆密码
	
	useradd git_admin -m //添加git_admin用户
	passwd git_admin //设置git_admin的登陆密码 

## 安装软件 ##
**安装Git**

	sudo apt-get update//更新软件源
	sudo apt-get install git
**生成sshkey**
切换到git_admin用户下，然后执行ssh-keygen生成sshkey，具体操作如下：

	sudo su git_admin//切换到git_admin
	ssh-keygen -t rsa -C "example@email.com"
这时候一直按回车就可以了，在你的根目录下就会生成.ssh文件夹，你可以通过**ls -al**来进行查看！将.ssh
文件夹下的id_rsa.pub文件拷贝到/tmp目录下备用。

```
cp ~/.ssh/id_rsa.pub /tmp
```

**安装gitolite**
切换到git用户，并新建bin目录，操作如下：

```
su git //切换到git用户
cd ~ //进入git用户目录
mkdir bin//新建bin文件夹'
```

下载gitolite代码

```
git clone http://github.com/sitaramc/gitolite
```

安装gitolite到bin目录

```
${HOME}/gitolite/install -to ${HOME}/bin
```

设置管理员ssh key

```
${HOME}/bin/gitolite setup -pk /tmp/id_rsa.pub
```

执行上面的步骤后，在git的根目录下会生成**projects.list**文件和**repositories**文件夹，projects.list就是描述仓库列表，repositories就是我们的项目仓库了。在repositories文件夹下面默认会有两个仓库，分别是gitolite-admin.git和testing.git。前者我们等下会讲到，利用gitolite-admin我们可以新增项目并管理用户，后者只是提供测试而已。

至此我们gitolite的安装其实已经完成了，那么下一步我们将介绍如何添加项目，用户以及配置项目用户权限。

## 管理员配置

切换到git_admin用户，然后将git用户下的git-admin.git仓库clone下来，具体操作如下：

```bash
sudo su git_admin//切换到git_admin用户
cd ~//进入到git_admin的用户目录/home/git_admin
git clone git@服务器ip地址(或者本机名称):gitolite-admin.git//clone gitolite-admin项目
```

在gitolite目录下存在**conf**与**keydir**两个目录，其中conf/下面有个名为gitolite.conf的配置文件。 

```
conf/gitolite.conf 用于Git项目配置，访问权限设置。
keydir/ 用于存储用户的SSH public key(公钥）。
```

管理员利用keydir可以增加与删除用户，利用conf就可以增加、删除项目并给用户配置不同的项目操作权限。

## 新增用户

如果要新增用户，该怎么操作呢？其实很简单，只需要用户通过ssh-keygen生成相应的公钥与密钥，然后将公钥交给我们的管理员，管理员修改公钥名称（为了区分不同的用户，因此需要重命名），最后放到keydir目录下就可以了。

```
用户生成公钥(id_rsa.pub)--->交给管理员---->修改名称(如：zhangsan.pub)--->放到keydir目录下--->提交到服务器（git commit and push）
```

## 新增项目

那如何新增项目呢？还记得默认有个testing的项目吗，我们完全可以依葫芦画瓢来新增项目。首先让我们来瞅一瞅conf/gitolite.conf文件：

```bash
repo gitolite-admin
    RW+     =   admin

repo testing
    RW+     =   @all 
```

R表示读权限，W表示写权限；@all表示所有的用户都对testing项目具有读写权限；因为我将git_admin的sshkey命名为admin.pub，所以gitolite-admin的读写用户为admin。这里我们可以得出结论，我们是根据keydir目录中的key的名称来区分不同的用户，比如zhangsan.pub放在keydir下，那么该用户名为zhangsan！！！！

假设我们要新增只有zhangsan才能读写的名为HelloWorld的项目，那么只需要这样写：

```
repo HelloWorld
	RW+     =   zhangsan
```

当你推送到远程仓库后，在git用户的repositories目录下就会新增一个HelloWorld.git远程仓库，张三可以通过git clone就可以将项目clone下来了。

```
git clone git@server_ip:HelloWorld.git
```

**注意：无论是你添加用户还是新增项目，你都需要git提交到远程仓库中。当你提交项目时，git用户目录下的repositories 就会新增该项目仓库。**

至此我们就利用gitolite简单的搭建了Git服务器进行用户与项目管理了，当然还有很多高深的配置与操作还有待去学习和发现。请参考官网：[Gitolite](http://gitolite.com/gitolite/index.html)