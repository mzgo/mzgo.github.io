---
date: '2026-04-09T17:02:40+08:00'
draft: false
title: '利用 Clash Verge 接管多个自研代理客户端'
---

# 技术笔记：利用 Clash Verge 接管多个自研代理客户端

## 场景描述

目前不少代理服务商（机场）为了规避风险或提供所谓“定制化服务”，不再直接提供通用的 Clash 订阅链接，而是强制使用其自研客户端。

怎么说呢，我可以理解，但很难接受，自研客户端可以放飞自我，用各种奇技淫巧去绕过运营商层面的封禁屏蔽，但是在某些大型会议期间，可能机场的稳定性统一都会变差，这时候能随时切换就很舒服，对于习惯了统一收口管理流量的开发者来说，在多个自研客户端之间切换非常低效。

本文记录如何通过配置 Clash Verge，将本地多个 SOCKS5 端口整合，实现统一的分流与调度。

## 核心思路：本地 Proxy Chaining
自研客户端在后台运行时，本质上是在 127.0.0.1 上开启了一个代理端口。我们可以通过 Clash 配置文件，将这些端口定义为 本地节点。

### 1. 配置逻辑（YAML 结构）
在 Clash Verge 中编辑你的配置文件（或通过 Merge 脚本注入），核心配置如下：

```yaml
proxies:
  - name: "Provider_A"
    type: socks5
    server: 127.0.0.1
    port: 6666  # 对应自研客户端A的本地端口

  - name: "Provider_B"
    type: socks5
    server: 127.0.0.1
    port: 6667  # 对应自研客户端B的本地端口
    
    
# 策略组，这里设置的是自动负载的
proxy-groups:
  - name: "Auto_Load_Balance"
    type: load-balance
    # 延迟测试地址，返回204表示连通性正常
    url: 'http://www.gstatic.com/generate_204'
    interval: 300
    proxies:
      - "Provider_A"
      - "Provider_B"

# 规则路由
rules:
  # 将全局匹配（或特定规则）指向你定义的策略组
  - MATCH, Auto_Load_Balance

```

### 2. 关于 clash-verge.yaml 的避坑说明
新手常犯的错误是直接去修改安装目录或 AppData 下的 clash-verge.yaml。

原因：该文件是 Clash Verge 的运行时生成的快照，每次启动或切换 Profile 时都会被覆盖。

正确做法：在订阅界面新建一个Local订阅文件，然后将上面的yaml配置填进去

### 3. 常见报错排查
为什么 UI 界面没有选项？
缩进错误：YAML 对空格极其敏感，确保 proxy-groups 与 proxies 同级缩进。

未激活配置：修改完 YAML 后，必须在 Profiles 页面重新点击 Use 才能触发内核重新加载。

## 总结
这种“收口”方式的好处在于：你可以利用 Clash 强大的 rules 分流功能（如域名分流、进程分流），去驱动那些原本功能单一的自研客户端。你只需要保证那些自研端在后台挂着，所有的流量管控都在 Clash Verge 这一个出口完成。

## 另外

顺带安利2个好用的东西

> SwitchyOmega chrome系浏览器代理切换插件

> Proxifier win/mac下可以让不支持配置代理的软件也能走代理
