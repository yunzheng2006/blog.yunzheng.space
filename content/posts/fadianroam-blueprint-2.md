---
title: 【FadianRoam发电漫游】蓝图笔记（2）— 联盟治理、网络分层和数据面设计
date: 2026-05-15T12:00:00+08:00
description: FadianRoam的治理设计、FadianNet数据面两个方案的对比、以及VLAN隔离和计费模型的思考
tags:
  - fadianroam
  - eduroam
draft: false
---

距离上一篇蓝图笔记过了几天，这几天一直在想FadianRoam到底应该怎么设计才合理，之前第一篇写的是头脑风暴阶段，现在终于有了一些比较完整的想法，记录一下。

## FadianRoam 和 FadianNet 到底是什么关系？

之前一直把这两个混在一起说，但是越想越觉得应该分开，因为它们本质上做的事完全不一样：

- **FadianRoam** — 就是认证漫游这一层，你的手机连上FadianRoam这个SSID，802.1X认证，RADIUS请求通过MGMT VPN走到Federation Relay再转发到你的Home Site验证身份，这整个过程就是FadianRoam干的事
- **FadianNet** — 认证通过之后你上网的数据要走哪里？走FadianNet。这是一个BGP骨干网，各个BGP站点用自己的ASN互联，提供实际的数据传输

所以FadianRoam运行在FadianNet之上，SSID、认证、项目品牌都是FadianRoam的，但是你实际上网的流量是FadianNet在承载。

而且FadianNet本身是公益的！BGP站点之间互联互通是大家一起维护的公共基础设施，不收费。但是FadianRoam作为业务层就不一样了，后面会说到。

## FadianNet的两种角色

想清楚分层之后，FadianNet的参与者其实分两种：

**FadianNet Site（服务商+使用者）**：有自己的ASN，跑BGP，可以给别人提供网络服务，同时自己也用FadianNet。这种就是骨干网的建设者。

**Access Member（纯使用者）**：没有ASN，不会BGP也没关系！通过FadianLink连到一个FadianNet Site上，拨PPPoE拿一个/32 IP，AP用户NAT出去就行了。

所以一个完整的FadianRoam站点，如果你同时能提供BGP网络服务，那你就是FadianRoam Site + FadianNet Site双重身份；如果你只是想部署AP让大家漫游上网，那你就是FadianRoam Site + Access Member。

## 联盟治理：>50% 投票制

这个是今天新想的，之前的加入流程太随意了——提交个PR维护者看看就合并了。但是这不对，FadianRoam是一个联盟，谁能加入应该是大家共同决定的。

新的流程：

1. 想加入的人提交PR（带上自己的RADIUS信息）
2. 所有现有的FadianRoam Site代表在PR上投票（GitHub Review: Approve/Request Changes）
3. **超过50%的现有成员批准**才能加入
4. 投票期3天

人少的时候也一样适用这个阈值，2个成员需要2票，3个成员需要2票。至少人少的时候我是这么想的。

等体系成熟了之后，由核心成员组建「发电委员会」（FadianRoam Governance Committee），负责组织运营、战略决策、争议仲裁这些事。

## VLAN隔离

每个FadianRoam站点的AP必须做VLAN隔离：

```
AP
 ├── SSID: FadianRoam ──→ VLAN 10 ──→ FadianNet (PPPoE /32 出网)
 └── SSID: 你自己的WiFi ──→ VLAN 20 ──→ 本地网络出口
```

为什么要隔离？因为FadianRoam的流量需要可计量、可审计。如果你把FadianRoam和自己家的WiFi混在一起走，那没法统计也没法做公平使用限制。而且安全上FadianRoam用户也不应该能访问你的本地网络。

## 计费和公平使用

说到计费，这个地方我想了挺久的。

首先明确一点：FadianRoam的精神就是**发电**——自己用、互相分享。所以计费系统**不是真的收费**，而是给大家划一个合理的爱好使用范围。

<img src="https://img.yunzheng.space/2026/05/bm6rsn6l.png" width="500">

等治理委员会成立之后会接入联盟计费系统，但这个系统只是**记录用量**，不直接扣费。联盟会给每个站点定一个基线（带宽、用户数、流量等），在基线内的使用完全免费，骨干网大家一起扛。

如果你的用量超过了这个爱好使用的基线，那需要你来**解释一下情况**——是临时的还是长期的？比如说你搞了个活动一天流量暴增，那很正常，说明一下就行。

有几种特殊场景是可以直接批准临时超额的，不需要额外费用：
- **FadianRoam自己的线下聚会**（成员meetup、技术交流）
- **跟FadianRoam合作的线下活动**（展会、演示、社区活动）
- **临时性的测试调试**（新站点上线、压力测试）

这些提前去 [YunZheng HelpCentre](https://helpdesk.yunzheng.space) 提个工单说明一下就好了。

真正需要讨论费用的只有一种情况：**商业使用**。如果你要拿FadianRoam做持续的商业WiFi服务、对外收费什么的，那确实需要承担对应的网络成本，这个需要在申请加入的时候就声明。

## 数据面设计：两个方案

这个是目前最纠结的地方，FadianNet的数据面到底怎么走，有两个方案：

<img src="https://img.yunzheng.space/2026/05/v08fhvd9.png" width="500">

### 方案A：共享公网/24前缀（Anycast模型）

我来sponsor一段/24，所有FadianNet Site用自己的ASN共同广播这段/24（RPKI多AS ROA），外部流量就近进入离用户最近的FadianNet Site（anycast），内部通过Regional RR eBGP传播/32细化路由做内部寻路。每个接入的站点通过PPPoE拿到一个/32 IP。

**好处**：统一地址空间、就近接入、延迟低、地址干净
**坏处**：需要找到一段/24、RPKI多AS ROA管理比较复杂

### 方案B：内网Loopback + 回Home上网

FadianNet内部用一个内网/24做loopback（类似OSPF那种），用户漫游之后流量隧道回到自己的Home Site上网。

**好处**：不需要sponsor公网前缀、每个站点用自己的IP资源
**坏处**：
1. **链路成本不均匀** — 你在亚洲连上的WiFi，流量要回到欧洲的Home Site再出网，延迟爆炸
2. **强制绑定BGP Player** — Access Member必须通过FadianNet Site做中转和回程，耦合太紧

目前倾向方案A，但是方案B也不是不行，看大家怎么想。欢迎来Telegram讨论：https://t.me/+WLLU-KOXcQFiMTg1

Telegram群里关于这个也有讨论，Jack（AS153376）提了个很好的问题：如果一个ORG的user特别多，比如JianyuelabLTD这种在各地都有很多用户连接到FadianRoam节点的，那这个站点到底是属于商用还是爱好使用？是不是要先考虑一下？

我的回复是：如果是商业情境，那委员会应该设立一个在FadianRoam上的使用标准，以「配额」为单位增加的同时，为所有BGP Sites均摊额外的成本（固定配额价值）。每个ORG产生的爱好配额以外的部分，由该ORG承担对应的BGP Sites付出的额外平摊成本。

## 用户注册

每个FadianRoam站点自己管自己的用户，自己决定开不开放注册。开放注册、邀请制、还是管理员手动创建，都行，联盟不强制。用户格式是 `用户名@站点realm`，比如 `edward.sun@roam.yunzheng.space`。

不管你在哪个站点注册的，连上任何一个FadianRoam的AP都能自动漫游认证。

## 最后

Blueprint网站已经更新了这些设计：[fadianroam.yunzheng.space](https://fadianroam.yunzheng.space)，而且加上了中英文切换。

目前YunZheng Lab的站点已经跑起来了（auth.yunzheng.space做IDP，FreeRADIUS做认证，OpenWrt AP），等第二个成员站点上线就可以测试跨站漫游了！

---

*本人因语言组合能力障碍，使用AI总结内容并在本文表述，如果遇到错误和建议，可以在评论提出，本文已经过发布者审核。*
