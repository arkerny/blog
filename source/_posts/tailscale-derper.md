---
title: 'arkerny Network NG Part.1 与自建 TailScale Derper 中继服务器'
date: 2026-10-17 14:53:59
categories: [技术]
tags: [TailScale, 'arkerny Network']
cover: 'https://split.img.arkerny.fun/images%2F2026%2F10-17-14-10-59-48ea3d8312f425b58c4de472198864dc-aa4ca2930f8613e09926539e551257cb_1792219501140_k6w6z9.webp'
description: '大内网时代'
---

## 大内网？

~~为了嫖取学校电费~~，自己手头上的各种 24h 服务放在了 702。同时直接新拉了个路由器接 iHDU。

同时因为各种原因，觉得利用 DN11 不是十分可行，~~让 cHAR0N 学习加入如何 DN11 疑似有点用力过猛了~~，于是没有选用大火都在用的看起来还不错的 DN11。

同时又有~~和 Shiori 分享百合漫画~~通过 DN11 访问部分服务的需求，以及 HDU 的神秘网络架构使我甚至无法在 Vidar Network 访问我的路由器（同层不通），显然还需要支持一部分的 DN11 访问。（这部分会在后面新开一篇 blog 写写）

以及由于神秘的 HDU 网络架构，即使在 HDU 内网中使用 Tailscale，也会出现打不通洞的情况。这时候自建的本地 Derper 服务器就显得尤为重要了。

之前的 arkerny Network 嘛，显然因为没钱交 ASN 和相关服务器的费用停了。也有了现在的 MoneyLess 的方案……

之前用的是 ZeroTier，但是近期感觉 Tailscale 更加好用（拉着 cHAR0N 一起建了个 Tailscale 网络）。Derper 也显然需要提上日程了。

目前要实现的就是和 cHAR0N 可以在各地（包括但不限于寝室、家中）使用较好的速度与延迟访问到各自的服务、以及（挂在 Vidar Network 上的）主机。

## 组网

于是首先是组网。

不赘述了，就是把身上所有东西上面装个 TailScale（x

## 部署 Derper

### 🐔？

选用了阿里云杭州的轻量应用服务器 2C 2G 200Mbps，价格中规中矩，可玩性不错。

200Mbps 的峰值带宽显然足够使用，最后实测速度有至少 50Mbps。

### 加入 TailScale

在服务器上运行 TailScale。

可以将服务器配置为 Exit Node，但是似乎没有什么卵用。

不如把学校的机子搞成 Exit Node，还能访问校内资源。

### Docker Compose

``` yaml
services:
  derper:
    image: fredliang/derper
    container_name: derper
    restart: always
    environment:
      - DERP_CERT_MODE=manual  # 使用手动证书
      - DERP_ADDR=:13477  # DERP 服务端口
      - DERP_HTTP_PORT=13476  # HTTP 端口
      - DERP_STUN_PORT=13478  # STUN 端口
      - DERP_DOMAIN=derp.arkerny.fun  # 替换为你的域名
      - DERP_VERIFY_CLIENTS=true  # 启用客户端验证
    ports:
      - "13477:13477"
      - "13476:13476"
      - "13478:13478/udp"
    volumes:
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock   #把外部的 Tailscale Socket 传到容器，很聪明的做法
      - ./cert:/app/certs
```

### SSL

用了 Let's Encrypt，目前方案是写了个脚本同步自动续订，但是随着平台、服务、SSL 证书的增多，正在寻求其他统一、优质的方法（x

### 阿里云的 ip 似乎冲突了

原因是 TailScale 防止有人在 TailScale 外伪造主机，便把非 TailScale 的网卡传入的 `100.64.0.0/10` 给 Drop 了。

那我们就直接 ACCEPT 阿里云的即可。

手动放行阿里云杭州内网 iptables
``` shell
sudo iptables -I ts-input 1 -s 100.100.2.136/32 -j ACCEPT   # DNS
sudo iptables -I ts-input 1 -s 100.100.2.138/32 -j ACCEPT   # DNS
sudo iptables -I ts-input 1 -s 100.100.2.148/32 -j ACCEPT   # Linux 镜像
sudo iptables -I ts-input 1 -s 100.100.100.200/32 -j ACCEPT # 监测服务（？）
sudo iptables -I ts-input 1 -s 100.100.45.106/32 -j ACCEPT  # 监测服务（？）
sudo iptables -I ts-input 1 -s 100.103.7.180/32 -j ACCEPT   # Docker 镜像
```

## 上线

在 [Tailscale Access Controls -> Json Editor](https://console.tailscale.com/admin/acls/file) 中加入自建的节点，（同时保留其余官方节点不动）：

``` json
"derpMap": {
    "OmitDefaultRegions": false,
    "Regions": {
      "900": {
        "RegionID": 900,
        "RegionCode": "hgh",
        "RegionName": "HangZhou",
        "Nodes": [
          {
            "Name": "Aliyun",
            "RegionID": 900,
            "HostName": "derp.arkerny.fun",
            "DERPPort": 13477,
            "STUNPort": 13478,
            "STUNOnly": false
          }
        ]
      }
    }
  }
```

检查下状态。

![](https://split.img.arkerny.fun/images%2F2026%2F10-17-15-10-46-ed2add306ff3b7f879892683643c52b9-ea7b76fa7698633068a149cade686aaa_1792222008808_bu6n08.webp)

那么就连上了

![](https://split.img.arkerny.fun/images%2F2026%2F10-17-15-10-53-7785b49e681d78da3681424360d6eaad-20261017154652519_1792223215969_nnbppp.webp)

速度不错

Done.