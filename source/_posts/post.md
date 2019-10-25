---
title: "腾讯云与GCP跨云服商VPC对联实践"
date: 2019-10-25 11:24:33
tags: 
- GCP 
- tencent 
- vpc
categories: 探索笔记
---

## 连通目标

受[aws和gcp vpc互通][site-to-site-vpn-between-gcp-and-aws]的启发，尝试使用类似的方式使GCP和腾讯云互通。本文测试环境为香港腾讯云和香港GCP.

## 需要用到的服务
  - GCP
     1. Hybrid Connectivity - VPN
     2. Compute Engine(用于测试连通性)
  - 腾讯云
     1. VPN网关
     2. VPN通道
     3. 对端网关
     4. cvm(用于测试连通性)

## 先决条件

腾讯云和GCP创建好用于连通的vpc

## 整体流程
  1. 腾讯云: 创建VPN网关
  2. GCP: 创建VPN gateway 并获取一个public static ip
  3. 腾讯云: 创建对端网关
  4. 腾讯云: 创建VPN通道
  5. GCP: 建立 VPN Connections
  6. 腾讯云: 更新路由表
  7. 测试连通

## 腾讯云-创建VPN网关

![腾讯云VPN网关](/kksnote2/images/tencent-vgw.png)
***所属网络为要对联的VPC***

## GCP-创建VPN gateway
  
Hybrid Connectivity -> VPN -> Cloud VPN Gateways -> Create VPN gateway. 由于似乎腾讯云不支持BGP，这里的VPN options我们选**Classic VPN**. 
  1. Reserve static ip
   ![Reserve static ip](/kksnote2/images/gcp-reserve-ip.png)
   这个ip记录下来，后面要用到。
  2. 腾讯云通道还没设置，下面的tunnels先删掉.
   ![delete tunnels](/kksnote2/images/gcp-tunnels.png)

## 腾讯云-创建对端网关

![对端网关](/kksnote2/images/tencent-op-vgw.png)
ip 为上一步**GCP的网关ip**

## 腾讯云-创建VPN通道

![第一步](/kksnote2/images/tencent-vpn-tunnel1.png)
![第二步](/kksnote2/images/tencent-vpn-tunnel2.png)
本地网段: **腾讯云VPC的CIDR**
对端网段: **GCP VPC的CIDR**

[腾讯官方文档][tencent-vpngw-doc].需要注意的是腾讯云只支持ikeV1,[腾讯][tencent-support-cipher]和[GCP][gcp-support-cipher]支持列表。综合起来设置为:
- 密钥: 自己生成记录下来，在GCP端需要用到.
- 加密算法: AES-128
- 认证算法: SHA1
- DH group: DH2
- 安全协议: ESP
- PFS: DH-GROUP2
![腾讯云ike](/kksnote2/images/tencent-ike.png)
![腾讯云ipsec](/kksnote2/images/tencent-ipsec.png)


## GCP: 建立 VPN Connections

Hybrid Connectivity -> VPN -> Cloud VPN Gateways -> 对应gateway -> Add VPN tunnel
  - Remote peer IP address : 腾讯云VPN网关ip
  - IKE pre-shared key: 上一步中自己生成的密钥
  - Routing options : Policy-based
  - Remote network IP range: 腾讯云VPN CIDR
  - Local subnetwork IP range: GCP要连通的CIDR

这一步结束两端状态应该如下:
    ![GCP状态](/kksnote2/images/gcp-tunnel-status.png)
    ![腾讯云状态](/kksnote2/images/tencent-tunnel-status.png)

## 腾讯云-更新路由表

私有网络 -> 路由表 -> 找到对应VPC路由表 -> 新增路由策略
  - 目的端: GCP VPC CIDR
  - 下一跳类型: VPN网关
  - 下一跳: VPN网关
  
## 测试连通

简单的测试方式可以登录其中一台机器,ssh username@ip的方式,ip为另一个VPC的测试机的vpc内ip,如果提示`Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`说明网络已通.

[site-to-site-vpn-between-gcp-and-aws]: https://medium.com/@oleg.pershin/site-to-site-vpn-between-gcp-and-aws-with-dynamic-bgp-routing-7d7e0366036d
[tencent-vpngw-doc]: https://cloud.tencent.com/document/product/554/18989
[tencent-support-cipher]: https://cloud.tencent.com/document/product/554/18904
[gcp-support-cipher]: https://cloud.google.com/vpn/docs/concepts/supported-ike-ciphers#phase-1_3