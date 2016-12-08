---
layout: post
title: "Linux服务器工具"
date: 2016-08-17 17:38:56 +0800
comments: true
categories: Linux服务器
---
## 集群间拷贝
在集群环境下，经常会遇到需要从一台服务器上向多个服务器拷贝软件包或者配置的情况，这时，pscp就派上用场了

<!--more-->

### 基本使用命令
    pscp -h hosts_file local_source_file remote_out_dir

其中的hosts_file内容如下

	host1:22
	host2:22
	...

### 其他参数
指定用户，使用 `-l username` 参数
传递目录，添加 `-r` 参数

