---
title: 【FadianRoam发电漫游】蓝图笔记（3）— SSID投票、数据面方案确定和社区讨论
date: 2026-05-16T12:00:00+08:00
description: SSID命名投票结果、数据面选定方案B（归属路由）、非BGP站点的赞助加入模型、464XLAT、以及群里关于peering和国内BGP的讨论
tags:
  - fadianroam
  - eduroam
draft: false
---

今天Telegram群里讨论了不少东西，而且数据面的方案也有了新的想法，记录一下。

## SSID命名投票

之前一直纠结SSID到底叫什么，是`FadianRoam`（大写R）还是`fadianroam`（全小写）还是`fadian_roam`（下划线），所以直接在群里开了个投票。

结果出来了：

- **FadianRoam** — 60%（3票）
- **fadianroam** — 40%（2票）
- **fadian_roam** — 0%
- **Other** — 0%

所以就定了，SSID统一用 `FadianRoam`，大写R。

<img src="https://img.yunzheng.space/2026/05/nzzpa1g3.png" width="500">

## 数据面方案：选了方案B

上一篇蓝图笔记说了两个方案还在纠结，今天想清楚了，倾向**方案B（内部环回 + 归属路由）**。

之前方案B的描述比较简单，现在完善一下设计：

### 核心思路

FadianNet所有站点**强制BGP**，没有BGP就不能成为FadianNet的骨干节点。大家用自己的ASN通过eBGP互联，组成一个虚拟的BGP骨干网。每个站点分配一个内部环回IP（类似OSPF的Loopback），/32路由在骨干网内通过Regional RR传播。

用户漫游之后，流量**路由回自己的归属站点**上网——就是你注册在哪个站点，流量就回到哪个站点出网。

```
FadianNet 骨干网（eBGP mesh）
│
├── 站点 A (AS204921) ← 自有互联网出口
│   ├── FadianRoam AP（自有用户 → 在 A 出网）
│   └── 非BGP站点 X（挂在 A 下面 → 也在 A 出网）
│
├── 站点 B (AS65001) ← 自有互联网出口
│   └── FadianRoam AP（自有用户 → 在 B 出网）
│
└── 站点 C (AS65002) ← 自有互联网出口
    └── FadianRoam AP（自有用户 → 在 C 出网）
```

### 漫游场景

**场景1**：站点A的用户跑到站点C连WiFi → 认证通过 → 数据从站点C走FadianNet骨干网回到站点A出网

**场景2**：非BGP站点X（挂在站点A下面）的用户跑到站点B连WiFi → 认证通过 → 数据从站点B走FadianNet骨干网回到站点A（赞助者）出网

### 为什么选方案B？

方案A（共享/24 Anycast）虽然延迟低，但需要sponsor一段/24前缀，还要搞RPKI多AS ROA，依赖性太强。方案B每个站点用自己的资源，强制BGP确保骨干网质量，而且流量统计很清晰——谁的用户谁的出口，一目了然。

当然方案B的缺点也明确：漫游延迟高，跨区域流量成本大。但对于社区网络来说，这个代价可以接受。两个方案都还在Blueprint上保留，欢迎讨论。

## 非BGP站点怎么加入？

这个是今天新设计的，跟RIPE的LIR/Enduser模型很像（只是类比，不是实际命名）：

没有BGP的站点不能直接加入FadianNet。要参与FadianRoam，你需要找到**你所在地区最近的一个FadianNet + FadianRoam站点**作为赞助者，然后：

1. 赞助者同意提供FadianLink连接
2. **赞助者代你提交申请PR**到联盟投票
3. 联盟投票（>50%，3天）——赞助者为你担保
4. 通过后，你通过FadianLink连接到赞助者，部署FadianRoam AP

你的所有用户流量都会通过FadianNet骨干网漫游回赞助者的出口上网。就像你是赞助者下面的一个"Enduser"一样。

## 464XLAT：没有公网IPv4也能玩

weitcis（KO6BBG / AS200825）提了一个很有意思的想法：如果站点没有公网IPv4，可以用464XLAT来解决。

简单来说464XLAT就是在IPv6-only的网络上跑IPv4，客户端把IPv4包装进IPv6（CLAT），服务端再解开（PLAT）。这样就算你的站点只有IPv6，用户设备照样能访问IPv4的互联网。

## Peering的热情

群里好几个人都在问peering的事，看起来大家对互联还挺积极的：

- 有人在CYVR、KSEA有节点
- RJTT也有人想peer
- 还有人在搞cloud init自动化部署公钥到服务器上

不过Yuuta（AS142281）问了个现实问题：在中国跑BGP是不是没法搞？这个确实是个问题，国内运营商环境比较特殊，不是随便就能起BGP的。

<img src="https://img.yunzheng.space/2026/05/58gmlpt4.png" width="500">

## 关于国内Peering

有人问FadianRoam的BGP出口都是国外的，那用FadianRoam上网就是走FadianNet的BGP Sites出去，像隧道一样？

是这样的。FadianRoam完全是一个国际化的项目，FadianNet的骨干网是各个BGP站点对外互联组成的虚拟网络。FadianRoam Site接入FadianNet就是使用FadianNet的BGP组网。流量通过骨干网漫游回你的归属站点出网，归属站点在哪里，你的出口就在哪里。

<img src="https://img.yunzheng.space/2026/05/ksf3yim0.png" width="500">

## RADIUS Proxy

群里还提到了一个接下来要研究的事：怎么做RADIUS proxy。这是FadianRoam的核心功能——当你在别人的站点连WiFi，本地RADIUS要把你的认证请求代理到你的Home Site去验证。这个Federation Relay的配置还需要大家一起研究完善。

## 最后

社区越来越活跃了，有人说「越来越好玩了」，确实是。从一开始就我一个人在想，到现在群里有好几个AS holder在讨论架构和peering，这种感觉挺好的。

欢迎更多人加入讨论：https://t.me/+WLLU-KOXcQFiMTg1

---

*本人患有注意力缺陷多动障碍（ADHD），在语言组织和长文写作方面存在困难，因此使用 AI 辅助总结和表述本文内容。如遇错误或有建议，欢迎在评论中提出。本文已经过发布者审核。*
