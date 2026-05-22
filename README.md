# 校园网安全接入与VPN远程接入系统

> 一个完整的校园网络工程项目，涵盖802.1x接入认证、DHCP安全防护、VPN远程接入、NAT地址转换与分层ACL访问控制。
>
> 设计阶段：优化方案设计 / 拓扑与编址规划阶段  
> 建设分期：一期（基础安全接入）+ 二期（扩展优化）

---

## 项目概述

本项目基于 **单核心三层架构**，设计并实现了一套完整的校园网安全接入系统。一期改造聚焦学生宿舍区的802.1x端口认证、DHCP Snooping安全防护、出口NAT与VPN远程接入，构建"准入前逻辑隔离、认证后按需放行"的安全接入体系。二期规划针对办公区与教学区提供Portal认证、PPPoE+BRAS、SDN-based接入控制三种替代方案的技术选型与架构设计。

### 核心技术栈

| 技术领域 | 具体实现 |
|---------|---------|
| **接入认证** | 802.1x Port-Based NAC（一期）/ Portal + MAB（二期推荐） |
| **VPN远程接入** | IPsec Site-to-Site VPN（家属区）+ PPTP Remote Access VPN（移动用户） |
| **地址分配** | 集中式DHCP Server + DHCP Relay（汇聚层代理） |
| **DHCP安全** | DHCP Snooping（Trusted/Untrusted端口 + 源MAC校验 + 地址绑定） |
| **地址转换** | NAPT（PAT Overload）+ NAT-Traversal（VPN穿越） |
| **访问控制** | 扩展ACL（核心层）+ 标准ACL（出口路由器VPN隔离） |

---

## 网络架构

### 拓扑结构

采用 **单核心三层架构**，核心层负责高速数据转发与策略控制，汇聚层实现区域流量汇聚与网关下沉，接入层提供终端接入与安全隔离。

```
                          ┌───────────────────────┐
                          │    ISP (模拟互联网)    │
                          │    Cisco 2621         │
                          │  Fa0/0: 111.100.0.6   │
                          │  Fa0/1: 202.115.20.1  │
                          │  S0/0:  202.115.21.1  │
                          └───────┬───────┬───────┘
                                  │       │
                     ┌────────────┘       └────────────┐
                     │                                 │
           ┌─────────▼─────────┐            ┌─────────▼─────────┐
           │  出口路由器 RSR20-24 │            │ 家属区网关 RSR20-14 │
           │  Fa0/0: 内网侧     │            │  S2/0: 公网侧      │
           │  Fa0/1: 111.100.0.5│            │  Fa0/1: 192.168.1.1│
           └─────────┬─────────┘            └───────────────────┘
                     │
           ┌─────────▼─────────┐
           │  核心交换机 S3760-24 │ ◄──── 三层交换 + DHCP Server + ACL
           │  Fa0/1: 172.16.101.1│       VLAN 10/31/32/33/40 网关
           │  Fa0/2: Trunk to C  │
           │  Fa0/3-5: 服务器区  │
           │  Fa0/24: 出口路由   │
           └────┬─────────┬─────┘
                │         │
    ┌───────────┘         └───────────┐
    │                                 │
┌───▼─────────────┐      ┌───────────▼──────────┐
│ 汇聚交换机 S3760-24│      │ C楼接入交换机 S2328G   │
│ Fa0/24: 172.16.101.2│      │ Fa0/1: Trunk to Core │
│ Fa0/1: Trunk to AB │      │ Fa0/1-24: 宿舍端口    │
│ VLAN 11/21/22 网关  │      │ 802.1x + DHCP Snooping│
│ DHCP Relay         │      └──────────────────────┘
└───┬─────────────┘
    │
┌───▼──────────────┐
│ AB楼接入 WS-C2912 │
│ Fa0/1: Trunk      │
│ Fa0/2-4: 办公A     │
│ Fa0/5-8: 教学B1    │
│ Fa0/9-12: 教学B2   │
└───────────────────┘
```

### 网络层次说明

| 层次 | 设备型号 | 物理位置 | 功能定位 |
|------|---------|---------|---------|
| **核心层** | 锐捷 S3760-24 | D楼网络中心 | 三层路由、DHCP Server、VLAN间路由、ACL控制 |
| **汇聚层** | 锐捷 S3760-24 | AB楼弱电间 | 区域汇聚、DHCP Relay、三层网关下沉 |
| **接入层-AB** | 思科 WS-C2912 | A+B楼 | 办公/教学区用户接入、端口VLAN划分 |
| **接入层-C** | 锐捷 S2328G | C楼宿舍区 | 宿舍用户接入、802.1x认证、DHCP Snooping |
| **出口层** | 锐捷 RSR20-24 | D楼边界 | NAT转换、IPsec/PPTP VPN、边界安全 |
| **Internet** | 思科 2621 | 模拟ISP | 公网路由模拟 |
| **家属区** | 锐捷 RSR20-14 | 家属区边界 | 站点NAT、IPsec VPN终结 |

---

## IP编址与VLAN规划

### VLAN分配表

| 区域 | VLAN ID | IP子网 | 信息点 | 备注 |
|------|---------|--------|--------|------|
| D楼服务器/网管 | 10 | 172.16.10.0/24 | 20 | 核心服务区，严格ACL隔离 |
| A楼办公区 | 11 | 172.16.11.0/24 | 120 | 独立广播域 |
| B1教学楼 | 21 | 172.16.21.0/24 | 100 | 独立广播域 |
| B2教学楼 | 22 | 172.16.22.0/24 | 100 | 独立广播域 |
| C1宿舍 | 31 | 172.16.31.0/24 | 80 | 启用802.1x + DHCP Snooping |
| C2宿舍 | 32 | 172.16.32.0/24 | 80 | 启用802.1x + DHCP Snooping |
| C3宿舍 | 33 | 172.16.33.0/24 | 80 | 启用802.1x + DHCP Snooping |
| RADIUS通信 | 40 | 172.16.40.0/30 | - | 管理平面，接入层到核心 |

### 互联地址规划

| 互联链路 | 地址段 | 说明 |
|---------|--------|------|
| 核心 ↔ 汇聚 | 172.16.101.0/30 | 三层点对点，核心.1 / 汇聚.2 |
| 核心 ↔ 出口路由 | 172.16.100.0/30 | 三层点对点，核心.1 / 路由.2 |
| 校园网出口 | 111.100.0.4/30 | 公网互联，出口.5 / ISP.6 |
| ISP ↔ 家属区 | 202.115.21.0/30 | 串口PPP，ISP.1 / 家属区.2 |
| ISP ↔ 移动用户 | 202.115.20.0/24 | 模拟公网用户接入 |
| 家属区内网 | 192.168.1.0/24 | 家属区局域网 |
| PPTP VPN | 10.200.200.0/24 | VPN拨入地址池，与172.16.0.0/16不重叠 |

---

## 项目目录结构

```
campus-network-security/
├── README.md                          # 项目总览（本文档）
├── docs/
│   ├── core-switch.md                 # D楼核心交换机 S3760-24 配置
│   ├── aggregation-switch.md          # AB楼汇聚交换机 S3760-24 配置
│   ├── access-switch-ab.md            # AB楼接入交换机 WS-C2912 配置
│   ├── access-switch-c.md             # C楼宿舍接入交换机 S2328G 配置
│   ├── exit-router.md                 # 出口路由器 RSR20-24 配置
│   ├── isp-router.md                  # ISP路由器 Cisco 2621 配置
│   ├── family-gateway.md              # 家属区路由器 RSR20-14 配置
│   ├── radius-server.md               # RADIUS/AAA服务器配置
│   └── verification-guide.md          # 分阶段验证测试指南
└── design-report.md                   # 完整设计报告（原始文档）
```

---

## 快速开始

### 部署顺序

按照以下顺序逐设备配置，每完成一个步骤后进行验证测试：

1. **步骤1：基础网络与VLAN配置**
   - 核心交换机 → 汇聚交换机 → 接入交换机（AB楼） → 接入交换机（C楼） → 出口路由器 → ISP路由器 → 家属区路由器
   - 验证：阶段0 基础网络连通性测试

2. **步骤2：DHCP服务部署**
   - 仅在核心交换机上配置集中式DHCP Server
   - 验证：阶段1 DHCP获取测试

3. **步骤3：DHCP Snooping与802.1x配置**
   - 仅在C楼接入交换机上配置DHCP Snooping和802.1x
   - 配合RADIUS服务器配置
   - 验证：阶段2 DHCP Snooping测试 → 阶段3 802.1x认证测试

4. **步骤4：NAT与VPN配置**
   - 出口路由器配置NAPT + IPsec VPN + PPTP VPN
   - 家属区路由器配置NAT + IPsec VPN
   - 验证：阶段4 NAT测试 → 阶段5 VPN测试

5. **步骤5：ACL配置**
   - 核心交换机配置宿舍区访问控制ACL
   - 出口路由器配置VPN用户隔离ACL

### 验证指南

详细的分阶段验证测试步骤请参考 [`docs/verification-guide.md`](docs/verification-guide.md)，包括：

- **阶段0**：基础网络连通性测试（VLAN内通信、跨VLAN路由、路由表完整性）
- **阶段1**：DHCP服务部署验证（自动获取、Relay转发、地址正确性）
- **阶段2**：DHCP Snooping验证（非法DHCP阻断、绑定表生成、源MAC校验）
- **阶段3**：802.1x认证验证（未认证隔离、合法认证、非法用户拒绝、单主机模式）
- **阶段4**：NAT部署验证（内网访问Internet、转换表生成、端口复用）
- **阶段5**：VPN部署验证（IPsec隧道建立、加密通信、PPTP拨入、NAT-Traversal）

---

## 版本变更记录

| 版本 | 日期 | 变更内容 | 适配器 |
|------|------|---------|--------|
| v5.0 | 2026/3 | 初始版本 | - |
| v5.1 | 2026/4/7 | 完成内部校园网连通性测试；修改思科二层交换机trunk编码；修改锐捷核心交换机上部分指令；统一将GigabitEthernet修改为FastEthernet | zql |
| v5.2 | 2026/4/20 | 修改ISP路由；修改出口路由器和家属区网关的NAT配置；修改IPsec VPN部分参数设置 | lg |
| v5.3 | 2026/4/26 | 将802.1x部分改为锐捷支持的命令；修改PPTP VPN配置部分指令以兼容锐捷RSR20系列 | zql |

---

## 技术文档索引

| 文档 | 内容说明 | 适用设备 |
|------|---------|---------|
| [core-switch.md](docs/core-switch.md) | VLAN创建、SVI三层接口、物理接口、静态路由、DHCP Server、ACL | 锐捷 S3760-24 |
| [aggregation-switch.md](docs/aggregation-switch.md) | VLAN创建、Trunk汇聚、SVI网关、DHCP Relay、静态路由 | 锐捷 S3760-24 |
| [access-switch-ab.md](docs/access-switch-ab.md) | VLAN数据库、Trunk上行、Access端口划分 | 思科 WS-C2912 |
| [access-switch-c.md](docs/access-switch-c.md) | VLAN创建、802.1x认证、DHCP Snooping、RADIUS对接 | 锐捷 S2328G |
| [exit-router.md](docs/exit-router.md) | NAT/PAT、IPsec Site-to-Site VPN、PPTP Remote Access VPN、ACL | 锐捷 RSR20-24 |
| [isp-router.md](docs/isp-router.md) | 公网接口配置、串口PPP、路由模拟 | 思科 2621 |
| [family-gateway.md](docs/family-gateway.md) | NAT配置、IPsec VPN、静态路由 | 锐捷 RSR20-14 |
| [radius-server.md](docs/radius-server.md) | NPS部署、RADIUS客户端配置、认证策略 | Windows Server |
| [verification-guide.md](docs/verification-guide.md) | 分阶段测试用例、验证命令、预期输出 | 全部设备 |

---

## 二期改造规划（选做）

针对办公区（A楼）和教学区（B1/B2楼）的高安全接入需求，推荐采用 **Portal认证结合MAC Authentication Bypass（MAB）的混合接入架构**：

- 利用现有锐捷S3760-24三层交换机内置的Web认证功能
- 通过HTTP/HTTPS重定向将未认证流量牵引至RADIUS服务器完成身份校验
- 基于Filter-Id属性动态下发ACL实现端口级访问控制
- 针对教学区多媒体设备启用MAB机制，以MAC地址为凭证自动完成认证
- 完全利旧现有二层/三层交换机与认证基础设施，无需采购新设备

---

## 已知问题与改进建议

1. **PPTP安全性**：PPTP未配置隧道分离（Split Tunneling）或ACL限制，默认PPTP用户获取地址后可访问完整内网，VPN账号泄露存在安全风险。

2. **认证体系不统一**：一期宿舍区使用802.1x，二期办公/教学区使用Portal+MAB，同一校园网存在两种认证体系，增加运维复杂度。

3. **设备兼容性**：锐捷S2328G老版本RGOS部分命令（如`host-mode`、`aaa authorization`）可能不支持，需根据实际版本调整。

---

## 实验环境

### 硬件设备

| 设备类型 | 型号/规格 | 数量 | 部署位置 |
|---------|----------|------|---------|
| 核心交换机 | 锐捷 S3760-24（三层） | 1台 | D楼网络中心 |
| 汇聚交换机 | 锐捷 S3760-24（三层） | 1台 | AB楼 |
| 接入交换机 | 锐捷 S2328G / 思科 WS-C2912 | 2台 | AB楼一台、C楼一台 |
| 出口路由器 | 锐捷 RSR20-24 | 1台 | 网络边界 |
| ISP路由器 | 思科 Cisco 2621 | 1台 | Internet中心 |
| 家属区网关 | 锐捷 RSR20-14 | 1台 | 家属区子网边界 |
| 认证服务器 | PC机 + Windows Server | 1台 | D楼 |
| 测试PC | 联想启天M695E（双网卡） | 8台 | 各区域模拟 |

### 软件系统

- 认证系统：Windows NPS（Network Policy Server）或 FreeRADIUS 3.x
- VPN客户端：Windows内置VPN客户端、Shrew Soft VPN Client（IPsec）
- 抓包分析：Wireshark（验证EAP协商、VPN隧道建立）

---

## License

本项目为教学实验用途，配置命令与设计方案供学习交流使用。
