---
title: OUC流量分析课-记一次中国海洋大学安全课程
author: dashjay
date: '2019-09-19'
slug: 'web-security-class-in-ouc'
categories: []
tags:
  - wireshark
  - aircrack-ng
type: ''
subtitle: ''
image: ''
---

最后修改与2020-12-31

说来惭愧，本人虽然是一个信息安全专业的人，而且目前已经是大学第三个年头了，可是我对安全方面的知识掌控能力基本为 0，因此完全可以当做一个小白来看待，本节课内容简单，记录为博客纯属凑数。

## 第一题

课程首先教了使用 `wireshark` 抓取 `ping` 产生的 `ICMP` `request` 和 `reply` 的包，目的应该是教会大家使用`wireshark` 的过滤条件

#### ICMP简介

> 什么是ICMP： ping 是基于ICMP工作的，全称 `Internet Control Message Protocol`，就是互联网控制报文协议。网络包在异常复杂的网络环境中传输时，常常会遇到各种各样的问题。当遇到问题的时候，要传出消息来，报告情况，这样才可以调整传输策略。

ICMP 请求包

<img src="/post/2019-09-19-ouc.zh_files/request.jpg"  align="center" width="700px">

ICMP 返回包
<img src="/post/2019-09-19-ouc.zh_files/reply.jpg"  align="center" width="700px">

看到了ping自己手机产生的ICMP包

![](https://img-blog.csdn.net/20180611011134623?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpYW9rdW56aGFuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 第二题

### (a)

分析题目提供的流量，解析获得FLAG

关键就是要一个一个找.
<img src="/post/2019-09-19-ouc.zh_files/follow_tcp.jpg" align="center" width="700px"/>

<img src="/post/2019-09-19-ouc.zh_files/base64_decode.jpg" align="center" width="700px"/>

## (b)

很快发现是flag是一张PNG

<img src="/post/2019-09-19-ouc.zh_files/find_by_http.jpg" align="center" width="700px"/>

将PNG包导出获得FLAG

## (c) 破解WPA加密的WIFI密码

使用命令配合github上的字典，快速破解得到密码

`$ aircrack-ng wifi.pcap -w ~/Desktop/my/Black/字典/fuzzDicts/passwordDict/top500.txt`

<img src="/post/2019-09-19-ouc.zh_files/aircrack.jpg" align="center" width="700px"/>

`$ airdecap-ng -e Blue-Whale -p 12345678 wifi.pcap` 指定密码将包解码

<img src="/post/2019-09-19-ouc.zh_files/airdecap.jpg" align="center" width="700px"/>

最后在读包过程中，找到FLAG

<img src="/post/2019-09-19-ouc.zh_files/find.jpg" align="center" width="700px"/>


愉快的课程就结束了，还是蓝鲸的研究生小姐姐带着做题的，很开心。
