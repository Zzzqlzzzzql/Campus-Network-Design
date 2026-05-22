# 汇聚交换机配置 — AB楼 S3760-24

> 设备型号：锐捷 S3760-24（三层交换机）  
> 部署位置：AB楼弱电间  
> 功能定位：区域汇聚、DHCP Relay代理、三层网关下沉  
> 主机名：`Aggre-AB-S3760`

---

## 接口分配

| 接口 | 类型 | 对端设备 | 说明 |
|------|------|---------|------|
| Fa0/1 | Trunk | AB楼接入交换机 WS-C2912 | 透传 VLAN 11/21/22 |
| Fa0/24 | 三层口 | D楼核心交换机 S3760-24 | 三层点对点 172.16.101.2/30 |

---

## 配置命令

### 1. 基础配置与VLAN创建

```shell
enable
configure terminal
hostname Aggre-AB-S3760

! 启用IP路由
ip routing

! === 本地VLAN创建 ===
vlan 11
 name Office_A
vlan 21
 name Teaching_B1
vlan 22
 name Teaching_B2
exit
```

### 2. 上行链路与路由配置

```shell
! === 上行链路：三层接口连接核心 ===
interface FastEthernet 0/24
 no switchport
 ip address 172.16.101.2 255.255.255.252
 no shutdown
 description To_Core_L3

! === 默认路由指向核心 ===
ip route 0.0.0.0 0.0.0.0 172.16.101.1
```

### 3. 下行Trunk配置

```shell
! === 下行Trunk连接AB楼接入交换机 ===
interface FastEthernet 0/1
 switchport mode trunk
 switchport allowed vlan add 11,21,22
 no shutdown
 description To_Access_AB
```

### 4. SVI三层网关配置（含DHCP Relay）

```shell
! === SVI作为各VLAN的三层网关 ===
! A楼办公区网关 + DHCP Relay
interface VLAN 11
 ip address 172.16.11.1 255.255.255.0
 description Office_A_Gateway
 no shutdown
 ip helper-address 172.16.101.1  ! DHCP Relay指向核心DHCP Server

! B1教学楼网关 + DHCP Relay
interface VLAN 21
 ip address 172.16.21.1 255.255.255.0
 description Teaching_B1_Gateway
 no shutdown
 ip helper-address 172.16.101.1

! B2教学楼网关 + DHCP Relay
interface VLAN 22
 ip address 172.16.22.1 255.255.255.0
 description Teaching_B2_Gateway
 no shutdown
 ip helper-address 172.16.101.1

end
write memory
```

---

## 技术说明

### DHCP Relay 工作原理

汇聚层交换机作为DHCP Relay Agent，将办公区和教学区的DHCP广播请求（Discover/Offer/Request/ACK）通过单播方式转发至核心层的DHCP Server（172.16.101.1），实现跨网段的集中式地址分配。

```
办公区PC(DHCP Discover) 
  → VLAN 11 广播
    → 汇聚交换机 SVI 11 接收
      → ip helper-address 172.16.101.1 单播转发
        → 核心交换机 DHCP Server 处理
          → 单播回应 DHCP Offer
            → 汇聚交换机 relay 回 VLAN 11
              → 办公区PC 收到地址分配
```

### 网关下沉设计

将办公区（VLAN 11）和教学区（VLAN 21/22）的三层网关下沉至汇聚层交换机，而非集中在核心层，有以下优势：

- **减轻核心负担**：区域间流量本地终结，无需经过核心交换机转发
- **提高转发效率**：同一区域内的流量在汇聚层即可完成三层转发
- **故障隔离**：单个区域的路由问题不影响其他区域和核心层

---

## 验证命令

```shell
! 查看VLAN信息
show vlan brief

! 查看Trunk链路状态
show interfaces trunk

! 查看三层接口状态
show ip interface brief

! 查看路由表
show ip route

! 查看DHCP Relay统计
show ip helper-address

! 查看SVI接口状态
show interface VLAN 11
show interface VLAN 21
show interface VLAN 22

! 保存配置
write memory
```

---

## 预期路由表

```
Gateway of last resort is 172.16.101.1 to network 0.0.0.0

C    172.16.11.0/24 is directly connected, VLAN11
C    172.16.21.0/24 is directly connected, VLAN21
C    172.16.22.0/24 is directly connected, VLAN22
C    172.16.101.0/30 is directly connected, FastEthernet0/24
S*   0.0.0.0/0 [1/0] via 172.16.101.1
```
