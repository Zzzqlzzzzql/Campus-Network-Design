# 出口路由器配置 — 锐捷 RSR20-24

> 设备型号：锐捷 RSR20-24  
> 部署位置：D楼网络边界  
> 功能定位：NAT地址转换、IPsec Site-to-Site VPN、PPTP Remote Access VPN、边界ACL  
> 主机名：`Exit-RSR20`

---

## 接口分配

| 接口 | IP地址 | 说明 |
|------|--------|------|
| Fa0/0 | 172.16.100.2/30 | 内网侧，连接核心交换机 |
| Fa0/1 | 111.100.0.5/30 | 外网侧，连接ISP |

---

## 配置命令

### 1. 基础配置

```shell
enable
configure terminal
hostname Exit-RSR20

! === 内网接口（连接核心） ===
interface FastEthernet 0/0
 ip address 172.16.100.2 255.255.255.252
 no shutdown
 description To_Core_Switch

! === 外网接口（连接ISP） ===
interface FastEthernet 0/1
 ip address 111.100.0.5 255.255.255.252
 no shutdown
 description To_ISP

! === 静态路由指向核心（汇总172.16.0.0/16） ===
ip route 172.16.0.0 255.255.0.0 172.16.100.1

! === 默认路由指向ISP ===
ip route 0.0.0.0 0.0.0.0 111.100.0.6

end
write memory
```

### 2. NAPT配置（PAT Overload）

```shell
enable
configure terminal

! === ACL定义：排除VPN地址池，允许内网 ===
! 排除PPTP VPN地址池（不参与NAT）
access-list 100 deny ip 10.200.200.0 0.0.0.255 any
! 排除IPsec VPN流量（直连家属区192.168.1.0/24）
access-list 100 deny ip 172.16.0.0 0.0.255.255 192.168.1.0 0.0.0.255
! 允许ICMP（Echo/Echo-Reply）用于Ping测试
access-list 100 permit icmp 172.16.0.0 0.0.255.255 any echo
access-list 100 permit icmp 172.16.0.0 0.0.255.255 any echo-reply
! 明确拒绝其他所有IP流量（阻断UDP等后台流量进入NAT）
access-list 100 deny ip 172.16.0.0 0.0.255.255 any

! === 接口NAT标识 ===
interface FastEthernet 0/0
 ip nat inside

interface FastEthernet 0/1
 ip nat outside

! === NAT地址池（仅一个公网地址，PAT端口复用） ===
ip nat pool NAT-POOL 111.100.0.5 111.100.0.5 netmask 255.255.255.255

! === PAT配置（端口多路复用，Many-to-One） ===
ip nat inside source list 100 pool NAT-POOL overload
```

### 3. IPsec Site-to-Site VPN（与家属区）

```shell
! === IKE策略（阶段1） ===
crypto isakmp policy 10
 encryption aes-256
 hash sha
 authentication pre-share
 group 2
 exit

! === 预共享密钥（对端为家属区公网地址） ===
crypto isakmp key 0 UESTC-IPSEC-KEY address 202.115.21.2

! === IPSec转换集（阶段2） ===
crypto ipsec transform-set UESTC-TS esp-aes-256 esp-sha-hmac
 mode tunnel

! === 感兴趣流ACL（定义需要加密的流量） ===
! 校园网内网 172.16.0.0/16 ↔ 家属区 192.168.1.0/24
access-list 101 permit ip 172.16.0.0 0.0.255.255 192.168.1.0 0.0.0.255

! === Crypto Map ===
crypto map UESTC-MAP 10 ipsec-isakmp
 set peer 202.115.21.2
 set transform-set UESTC-TS
 match address 101

! === 应用到外网接口 ===
interface FastEthernet 0/1
 crypto map UESTC-MAP

! NAT-T默认启用，无需手动配置
! 当检测到NAT设备时，ESP自动封装在UDP 4500端口
```

### 4. PPTP Remote Access VPN（移动用户）

```shell
! === 启用VPDN（PPTP） ===
! 【v5.3】使用 vpdn enable 替代旧的 pptp enable
vpdn enable

! === 创建VPDN组 ===
! 【v5.3】新增配置方式
vpdn-group PPTP-Group
 accept-dialin protocol pptp virtual-template 1

! === 虚拟模板接口 ===
interface Virtual-Template 1
 ip address 10.200.200.1 255.255.255.0
 encapsulation ppp
 ! 认证方式：MS-CHAPv2
 ppp authentication ms-chap-v2
 ! 【v5.3】MPPE加密命令格式
 ppp mppe default
 ! 分配地址池
 peer default ip address pool PPTP-Pool

! === 地址池配置 ===
! 【v5.3】起始地址改为 10.200.200.2
ip local pool PPTP-Pool 10.200.200.2 10.200.200.254

! === PPP认证配置 ===
! 方式A：通过AAA认证（推荐）
aaa authentication ppp default local
! 【v5.3】若上条命令报错，使用方式B：
! 方式B：直接在VT接口下配置本地认证
! interface virtual-template 1
!  ppp authentication ms-chap-v2 local

! === 本地用户数据库 ===
username vpnuser password UESTC@2026 service-type ppp

end
write memory
```

### 5. VPN用户ACL（隔离控制）

```shell
enable
configure terminal

! === ACL 110：限制PPTP VPN用户访问范围 ===
! 禁止VPN用户访问服务器区（高敏感区域）
access-list 110 deny ip 10.200.200.0 0.0.0.255 172.16.10.0 0.0.0.255
! 允许VPN用户访问内网其他区域
access-list 110 permit ip 10.200.200.0 0.0.0.255 172.16.0.0 0.0.255.255
! 拒绝VPN用户访问其他（Internet等）
access-list 110 deny ip 10.200.200.0 0.0.0.255 any

! === 应用到VT接口入方向 ===
! 【v5.3】使用 virtual-template 1 替代 FastEthernet 0/0
interface virtual-template 1
 ip access-group 110 in

end
write memory
```

---

## VPN技术详解

### IPsec Site-to-Site VPN 架构

```
校园网出口(111.100.0.5)              Internet               家属区网关(202.115.21.2)
┌─────────────────┐                  ┌──────────┐          ┌─────────────────┐
│  crypto isakmp  │◄── IKE协商 ─────►│   ISP    │◄────────►│  crypto isakmp  │
│  policy 10      │   (UDP 500)      │  2621    │          │  policy 10      │
├─────────────────┤                  ├──────────┤          ├─────────────────┤
│  PSK: UESTC-IP  │◄── IPsec隧道 ──►│ 透明转发  │◄────────►│  PSK: UESTC-IP  │
│  SEC-KEY        │   (ESP/UDP4500)  │          │          │  SEC-KEY        │
├─────────────────┤                  └──────────┘          ├─────────────────┤
│  172.16.0.0/16  │◄──── 加密流量 ────────────────────────►│  192.168.1.0/24 │
│  ←─────────→    │     (AES-256 + SHA-HMAC)                ←─────────→      │
└─────────────────┘                                        └─────────────────┘
```

### PPTP Remote Access VPN 架构

```
校外移动用户                       Internet                      校园网出口
(Windows/macOS)                  (ISP 202.115.20.0/24)          (RSR20-24)
┌─────────────┐                  ┌──────────┐                  ┌─────────────┐
│ PPTP客户端   │◄─ TCP 1723 ────►│   ISP    │◄────────────────►│ VPDN Enable │
│ 内置拨号     │   控制连接        │  2621    │                  │ PPTP-Group  │
├─────────────┤                  ├──────────┤                  ├─────────────┤
│ PPP over    │◄─ GRE协议 ──────►│ 透明转发  │◄────────────────►│ Virtual-    │
│ GRE封装     │   数据通道        │          │                  │ Template 1  │
├─────────────┤                  └──────────┘                  ├─────────────┤
│ 获取10.200. │◄─ MPPE加密 ──────────────────────────────────►│ 10.200.200.0│
│ 200.x地址   │     (RC4-128)                                   /24 地址池    │
└─────────────┘                                                └─────────────┘
```

---

## 验证命令

```shell
! === NAT相关 ===
! 查看NAT转换表
show ip nat translations
! 查看NAT统计
show ip nat statistics
! 查看详细NAT转换信息
show ip nat translations verbose

! === IPsec VPN相关 ===
! 查看IKE SA
show crypto isakmp sa
! 查看IPsec SA
show crypto ipsec sa
! 查看Crypto Map状态
show crypto map

! === PPTP VPN相关 ===
! 查看VPDN状态
show vpdn
! 查看PPTP会话
show vpdn session
! 查看PPTP隧道
show vpdn tunnel
! 查看登录用户
show users

! === 路由相关 ===
! 查看路由表
show ip route
! 查看接口状态
show ip interface brief

! 保存配置
write memory
```

---

## 预期输出

### NAT转换表示例

```
Pro  Inside global      Inside local        Outside local       Outside global
---  111.100.0.5:1025   172.16.11.10:5000   111.100.0.6:53      111.100.0.6:53
---  111.100.0.5:1026   172.16.21.10:5001   111.100.0.6:53      111.100.0.6:53
---  111.100.0.5:1027   172.16.31.10:5002   111.100.0.6:53      111.100.0.6:53
```

### IKE SA状态示例

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id slot status
111.100.0.5     202.115.21.2    QM_IDLE           1001    0 ACTIVE
```
