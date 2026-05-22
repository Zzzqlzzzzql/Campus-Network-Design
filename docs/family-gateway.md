# 家属区路由器配置 — 锐捷 RSR20-14

> 设备型号：锐捷 RSR20-14  
> 部署位置：家属区子网边界  
> 功能定位：家属区NAT转换、IPsec Site-to-Site VPN终结  
> 主机名：`Family-Gateway-RSR20-14`

---

## 接口分配

| 接口 | IP地址 | NAT角色 | 说明 |
|------|--------|---------|------|
| S2/0 | 202.115.21.2/30 | outside | 连接ISP（公网侧，串口PPP） |
| Fa0/1 | 192.168.1.1/24 | inside | 连接家属区内网（私网侧） |

---

## 配置命令

### 1. 基础配置

```shell
enable
configure terminal
hostname Family-Gateway-RSR20-14

! === 连接ISP（串口，公网侧） ===
interface Serial 2/0
 ip address 202.115.21.2 255.255.255.252
 encapsulation ppp
 ip nat outside
 no shutdown
 description To_ISP_Serial

! === 连接家属区内网（私网侧） ===
interface FastEthernet 0/1
 ip address 192.168.1.1 255.255.255.0
 ip nat inside
 no shutdown
 description To_Family_LAN

! === 指向校园网的路由 ===
ip route 172.16.0.0 255.255.0.0 202.115.21.1

! === 默认路由指向ISP ===
ip route 0.0.0.0 0.0.0.0 202.115.21.1

end
write memory
```

### 2. NAT配置

```shell
enable
configure terminal

! === ACL：控制NAT转换流量 ===
! 拒绝去往校园网的流量（保留私网地址供IPsec匹配）
access-list 100 deny ip 192.168.1.0 0.0.0.255 172.16.0.0 0.0.255.255
! 允许ICMP（Echo/Echo-Reply）用于Ping测试
access-list 100 permit icmp 192.168.1.0 0.0.0.255 any echo
access-list 100 permit icmp 192.168.1.0 0.0.0.255 any echo-reply
! 明确拒绝其他所有IP流量（阻断UDP等后台流量进入NAT）
access-list 100 deny ip 192.168.1.0 0.0.0.255 any

! === NAT地址池（PAT端口复用） ===
ip nat pool NAT-POOL 202.115.21.2 202.115.21.2 netmask 255.255.255.255

! === NAT转换（PAT Overload） ===
ip nat inside source list 100 pool NAT-POOL overload

end
write memory
```

> **重要**：NAT ACL中明确拒绝了 `192.168.1.0/24 → 172.16.0.0/16` 的流量，确保访问校园网的流量**不会被NAT转换**，而是以原始私网地址 `192.168.1.x` 进入IPsec隧道，匹配感兴趣流 `192.168.1.0/24 ↔ 172.16.0.0/16`。

### 3. IPsec VPN配置

```shell
enable
configure terminal

! === IKE策略（与出口路由器完全对应） ===
crypto isakmp policy 10
 encryption aes-256
 hash sha
 authentication pre-share
 group 2

! === 预共享密钥（对端为校园网出口公网地址） ===
crypto isakmp key 0 UESTC-IPSEC-KEY address 111.100.0.5

! === 转换集（与出口路由器完全对应） ===
crypto ipsec transform-set UESTC-TS esp-aes-256 esp-sha-hmac
 mode tunnel

! === 感兴趣流（与出口路由器镜像） ===
! 家属区 192.168.1.0/24 ↔ 校园网 172.16.0.0/16
access-list 101 permit ip 192.168.1.0 0.0.0.255 172.16.0.0 0.0.255.255

! === Crypto Map ===
crypto map UESTC-MAP 10 ipsec-isakmp
 set peer 111.100.0.5
 set transform-set UESTC-TS
 match address 101

! === 应用到串口（公网接口） ===
interface Serial 2/0
 crypto map UESTC-MAP

end
write memory
```

---

## IPsec VPN 配置对照表

两侧配置必须严格对应，任何不匹配都会导致隧道协商失败：

| 参数 | 校园网出口 RSR20-24 | 家属区网关 RSR20-14 |
|------|-------------------|-------------------|
| **IKE加密** | aes-256 | aes-256 |
| **IKE哈希** | sha | sha |
| **IKE认证** | pre-share | pre-share |
| **DH组** | group 2 | group 2 |
| **预共享密钥** | UESTC-IPSEC-KEY | UESTC-IPSEC-KEY |
| **对端地址** | 202.115.21.2 | 111.100.0.5 |
| **ESP加密** | aes-256 | aes-256 |
| **ESP认证** | sha-hmac | sha-hmac |
| **模式** | tunnel | tunnel |
| **感兴趣流（本地→远端）** | 172.16.0.0/16 → 192.168.1.0/24 | 192.168.1.0/24 → 172.16.0.0/16 |

---

## 验证命令

```shell
! === 接口与路由 ===
! 查看接口状态
show ip interface brief
! 查看路由表
show ip route

! === NAT相关 ===
! 查看NAT转换表
show ip nat translations
! 查看NAT统计
show ip nat statistics

! === IPsec VPN相关 ===
! 查看IKE SA
show crypto isakmp sa
! 查看IPsec SA
show crypto ipsec sa
! 查看Crypto Map
show crypto map

! === 连通性测试 ===
! 测试与ISP连通性
ping 202.115.21.1
! 测试与校园网出口连通性（经IPsec隧道）
ping 172.16.10.10 source 192.168.1.1

! 保存配置
write memory
```

---

## 预期输出

### IKE SA状态

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id slot status
202.115.21.2    111.100.0.5     QM_IDLE           1001    0 ACTIVE
```

### IPsec SA状态（双向）

```
interface: Serial2/0
    Crypto map tag: UESTC-MAP, local addr 202.115.21.2

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (192.168.1.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (172.16.0.0/255.255.0.0/0/0)
   current_peer 111.100.0.5 port 500
     PERMIT, flags={origin_is_acl,}
     #pkts encaps: 10, #pkts encrypt: 10, #pkts digest: 10
     #pkts decaps: 10, #pkts decrypt: 10, #pkts verify: 10
```

---

## 故障排查

| 问题 | 可能原因 | 排查命令 |
|------|---------|---------|
| IPsec隧道无法建立 | IKE策略不匹配 | `show crypto isakmp sa` |
| 感兴趣流不触发加密 | ACL配置错误/镜像不一致 | `show access-list 101` |
| 家属区无法访问校园网 | 路由缺失 | `show ip route 172.16.0.0` |
| NAT错误转换VPN流量 | NAT ACL顺序问题 | `show ip nat translations` |
| PPP链路down | 封装类型或时钟问题 | `show interface s2/0` |
| 加密但不通 | 防火墙阻断ESP/GRE | `debug crypto ipsec` |
