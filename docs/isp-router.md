# ISP路由器配置 — Cisco 2621

> 设备型号：思科 Cisco 2621  
> 部署位置：Internet中心（模拟）  
> 功能定位：模拟Internet骨干网、公网路由、连接移动用户与家属区  
> 主机名：`ISP-Simulator`

---

## 接口分配

| 接口 | IP地址 | 说明 |
|------|--------|------|
| Fa0/0 | 111.100.0.6/30 | 连接校园网出口路由器 RSR20-24 |
| Fa0/1 | 202.115.20.1/24 | 连接移动用户（模拟公网用户接入） |
| S0/0 | 202.115.21.1/30 | 连接家属区路由器 RSR20-14（串口PPP） |

---

## 配置命令

### 基础配置

```bash
enable
configure terminal
hostname ISP-Simulator

! === 连接校园网出口 ===
interface FastEthernet0/0
 ip address 111.100.0.6 255.255.255.252
 no shutdown
 description To_Campus_Exit

! === 连接移动用户 ===
interface FastEthernet0/1
 ip address 202.115.20.1 255.255.255.0
 no shutdown
 description To_Mobile_User

! === 连接家属区（串口） ===
interface Serial0/0
 ip address 202.115.21.1 255.255.255.252
 encapsulation ppp
 clock rate 64000
 no shutdown
 description To_Family_Gateway_WAN

end
write memory
```

---

## 技术说明

### 透明转发设计

ISP路由器在本项目中**仅做透明路由转发**，不需要配置NAT、ACL或VPN。其作用是：

1. **模拟公网环境**：提供校园网出口（111.100.0.4/30）与家属区（202.115.21.0/30）之间的可达性
2. **承载IPsec隧道**：校园网出口（111.100.0.5）与家属区网关（202.115.21.2）之间的IKE/IPsec流量经此设备透明转发
3. **承载PPTP连接**：移动用户（202.115.20.0/24）与校园网出口（111.100.0.5）之间的PPTP控制连接（TCP 1723）和数据通道（GRE协议号47）经此设备转发

### 路由说明

ISP路由器依靠直连路由完成转发：
- `111.100.0.4/30` → Fa0/0（校园网出口侧）
- `202.115.20.0/24` → Fa0/1（移动用户侧）
- `202.115.21.0/30` → S0/0（家属区侧）

由于三个接口分别属于不同子网，无需额外配置静态路由即可实现三者之间的互联互通。

### 串口配置要点

| 参数 | 值 | 说明 |
|------|------|------|
| 封装协议 | PPP | 点到点协议 |
| 时钟频率 | 64000 bps | DCE端提供时钟（ISP侧为DCE） |
| IP地址 | 202.115.21.1/30 | 与家属区网关202.115.21.2点对点互联 |

---

## 验证命令

```shell
! 查看接口状态
show ip interface brief

! 查看路由表（应只有直连路由）
show ip route

! 验证与校园网出口连通性
ping 111.100.0.5

! 验证与家属区网关连通性
ping 202.115.21.2

! 查看串口状态
show interface serial0/0

! 查看PPP协商状态
show interface serial0/0 | include PPP

! 保存配置
write memory
```

---

## 预期路由表

```
C    111.100.0.4/30 is directly connected, FastEthernet0/0
C    202.115.20.0/24 is directly connected, FastEthernet0/1
C    202.115.21.0/30 is directly connected, Serial0/0
```

---

## 故障排查

| 问题 | 可能原因 | 排查命令 |
|------|---------|---------|
| 校园网无法访问家属区 | 接口down | `show ip interface brief` |
| IPsec隧道无法建立 | ISP未转发UDP 500/4500 | `show interface fa0/0` |
| PPTP连接失败 | GRE协议被丢弃 | `debug ip packet`（临时） |
| 串口协议down | 时钟频率未配置 | `show interface s0/0` |
| PPP协商失败 | 封装类型不匹配 | `show interface s0/0` |
