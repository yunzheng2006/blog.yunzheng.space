---
title: 【手搓运营商】自己在家手搓运营商的记录1
date: 2026-05-14T12:15:19+08:00
description: 20260514-RR设计的思索
tags:
  - fadianroam
  - eduroam
draft: false
---

这次是我写的第二个博客文章，今天主要检索了一下心中对于网络的设计的规划，AS204921 v2 指日可待。

在v1的时候，每一个节点都是手动配置的文件，虽然配置文件都是一套，但是不同的节点还是需要改很多地方，甚至创建Session的时候都是单独复制一遍之前同类型的Session改endpoint，并且之前用的是完全静态的过滤器，每一个Session都需要单独配置，十分麻烦！

在芸峥实验网v2的环境下，上线了兼容bird（gobgp agent）和Mikrotik（ROS RestAPI）节点管理的配置下发面板（当然是vibe出来的），彻底托管配置，无需手动编辑，可以自定义所有Session以及单独的过滤器和RTBH策略等，所以自动化虚拟骨干网指日可待！

<img src="https://img.yunzheng.space/2026/05/3etk4z53.png" width="800">

在v1网络中，之前的经验是FullMesh+静态IBGP，每一个端口的endpoint都要配置静态保护路由，才会保证不会掉线，十分不灵活，之前改ospf cost还需要用脚本去检测，但是v2网络中就设计了自动化流程，十分方便。

## 一些理解

但是在之前v1的网络中，我似乎有一个理解错误，那就是 “使用Tunnel LAN 做Peering Endpoint（nexthop）”，这样就丧失了OSPF的意义，因为OSPF的定位就是IGP Peering中传递 **内部路由（内网地址）** 的协议，也就是说 会有一个 LoopBack CIDR用于 OSPF内部传播，比如YunZhengNet v2 在LoopBack使用的prefix是 10.204.0.0/24 ，那么这一段就是OSPF的Whitelist（个人理解），然后每个router通过OSPF传播自己的 /32，如果在OSPF Mesh中，某个pop上线了传递自己的/32，那就相当于全部成员都有着一条路由表的不同路线。

参考了一直以来的挚友的建议，她说IBGP Session应该建立到OSPF Pop Endpoint上面，而不是Tunnel Endpoint，这样才能是动态的传递IBGP路由（因为链路层就是IGP的动态），这样似乎才是合理的，**OSPF Loopback as Multihop IBGP PeeringLAN**(Mabe?)

顺便之前IBGP起来了之后，不论我加上保护路由，机器还是会掉线，起初我还以为是机器问题，后来才发现，先简历保护路由后起IBGP，因为IBGP也跑的事FullTable所以导入内核的路由就会影响之前配置的静态Endpoint，导致短路。所以新的方法还是稳妥的跟bird一样导入不同table（bird目前是bgp在main，保护路由在table1000）ros同理。

## 有趣的事

不过有一个有趣的点，因为我一直使用RestAPI下发配置给ROS，所以ROS那边会出现下发成功或者下发到一半的时候突然卡住，但是ssh能连上，webui和winbox全都卡住，让我迷惑了很久；询问了Jasper Yu大佬之后，大佬说可能是机器硬件问题maybe？（他不知道我用rest API），我也一度以为是机器硬件有什么问题，直到vibe过程中codex告诉我：是因为**添加静态路由的时候会lookup全表，导致webserver拖炸了**））如果你们之前也遇到过这样的问题希望给你们解决思路;;;

<img src="https://img.yunzheng.space/2026/05/w2vx8mst.png" width="400">

这是我写的第二篇blog，我不会写，希望各位给予评价和建议，谢谢！