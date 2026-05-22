# 核心交换机配置 — D楼 S3760-24

> 设备型号：锐捷 S3760-24（三层交换机）  
> 部署位置：D楼网络中心  
> 功能定位：核心交换、DHCP Server、VLAN间路由、ACL访问控制  
> 主机名：`Core-D-S3760`

---

## 接口分配

| 接口 | 类型 | 对端设备 | 说明 |
|------|------|---------|------|
| Fa0/1 | 三层口 | AB楼汇聚交换机 | 三层点对点 172.16.101.1/30 |
| Fa0/2 | Trunk | C楼接入交换机 S2328G | 透传 VLAN 31/32/33/40 |
| Fa0/3-5 | Access VLAN 10 | 服务器群 | 直连服务器区 |
| Fa0/24 | 三层口 | 出口路由器 RSR20-24 | 三层点对点 172.16.100.1/30 |

---

## 配置命令

### 1. 全局配置与VLAN创建

```shell
enable
configure terminal
hostname Core-D-S3760

! 启用IP路由
ip routing

! === VLAN规划 ===
vlan 10
 name Server_Mgmt
vlan 31
 name Dorm_C1
vlan 32
 name Dorm_C2
vlan 33
 name Dorm_C3
vlan 40
 name Control_Plane
exit
```

### 2. 三层接口配置（SVI）

```shell
! === 服务器区网关 ===
interface VLAN 10
 ip address 172.16.10.1 255.255.255.0
 description Server_Farm_Gateway
 no shutdown

! === 宿舍区网关 ===
interface VLAN 31
 ip address 172.16.31.1 255.255.255.0
 description Dorm_C1_Gateway
 no shutdown

interface VLAN 32
 ip address 172.16.32.1 255.255.255.0
 description Dorm_C2_Gateway
 no shutdown

interface VLAN 33
 ip address 172.16.33.1 255.255.255.0
 description Dorm_C3_Gateway
 no shutdown

! === 管理VLAN（用于接入层交换机管理与RADIUS通信） ===
interface VLAN 40
 ip address 172.16.40.1 255.255.255.252
 description Mgmt_for_Access_Switches
 no shutdown
```

### 3. 物理接口配置

```shell
! === 连接服务器群（Fa0/3-5） ===
interface range FastEthernet 0/3-5
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
 description Server_Farm_Direct_Connect

! === 连接C楼接入交换机（Trunk） ===
interface FastEthernet 0/2
 switchport mode trunk
 switchport trunk allowed vlan add 31,32,33,40
 switchport trunk native vlan 1
 no shutdown
 description To_Dorm_C_Access

! === 连接AB楼汇聚交换机（三层点对点） ===
interface FastEthernet 0/1
 no switchport
 ip address 172.16.101.1 255.255.255.252
 no shutdown
 description To_Aggre_AB_L3

! === 连接出口路由器（三层） ===
interface FastEthernet 0/24
 no switchport
 ip address 172.16.100.1 255.255.255.252
 no shutdown
 description To_Exit_Router
```

### 4. 静态路由配置

```shell
! 缺省路由指向出口路由器
ip route 0.0.0.0 0.0.0.0 172.16.100.2

! 指向汇聚层的路由（办公区/教学区网段）
ip route 172.16.11.0 255.255.255.0 172.16.101.2
ip route 172.16.21.0 255.255.255.0 172.16.101.2
ip route 172.16.22.0 255.255.255.0 172.16.101.2

! PPTP VPN用户网段回程路由
ip route 10.200.200.0 255.255.255.0 172.16.100.2

end
write memory
```

### 5. DHCP服务配置

```shell
enable
configure terminal

! 启用DHCP服务
service dhcp

! === 办公区地址池（A楼，通过Relay分配） ===
ip dhcp pool Office_Pool
 network 172.16.11.0 255.255.255.0
 default-router 172.16.11.1
 dns-server 172.16.10.10 8.8.8.8
 lease 0 8 0

! === 教学区地址池（B1楼） ===
ip dhcp pool Teaching_B1_Pool
 network 172.16.21.0 255.255.255.0
 default-router 172.16.21.1
 dns-server 172.16.10.10 8.8.8.8

! === 教学区地址池（B2楼） ===
ip dhcp pool Teaching_B2_Pool
 network 172.16.22.0 255.255.255.0
 default-router 172.16.22.1
 dns-server 172.16.10.10 8.8.8.8

! === 宿舍区地址池（C1，租期4小时） ===
ip dhcp pool Dorm_C1_Pool
 network 172.16.31.0 255.255.255.0
 default-router 172.16.31.1
 dns-server 172.16.10.10 8.8.8.8
 lease 0 4 0

ip dhcp pool Dorm_C2_Pool
 network 172.16.32.0 255.255.255.0
 default-router 172.16.32.1
 dns-server 172.16.10.10 8.8.8.8
 lease 0 4 0

ip dhcp pool Dorm_C3_Pool
 network 172.16.33.0 255.255.255.0
 default-router 172.16.33.1
 dns-server 172.16.10.10 8.8.8.8
 lease 0 4 0

! === 排除网关地址（不参与分配） ===
ip dhcp excluded-address 172.16.11.1
ip dhcp excluded-address 172.16.21.1 172.16.21.10
ip dhcp excluded-address 172.16.22.1 172.16.22.10
ip dhcp excluded-address 172.16.31.1 172.16.31.10
ip dhcp excluded-address 172.16.32.1 172.16.32.10
ip dhcp excluded-address 172.16.33.1 172.16.33.10

end
write memory
```

### 6. ACL访问控制配置

```shell
enable
configure terminal

! === 创建扩展ACL：限制宿舍区访问服务器区 ===
ip access-list extended DORM_TO_SERVER
 ! 允许访问服务器区HTTP（图书馆Web）
 permit tcp 172.16.31.0 0.0.0.255 172.16.10.0 0.0.0.255 eq 80
 permit tcp 172.16.32.0 0.0.0.255 172.16.10.0 0.0.0.255 eq 80
 permit tcp 172.16.33.0 0.0.0.255 172.16.10.0 0.0.0.255 eq 80
 ! 拒绝其他对服务器区的访问
 deny ip 172.16.31.0 0.0.0.255 172.16.10.0 0.0.0.255
 deny ip 172.16.32.0 0.0.0.255 172.16.10.0 0.0.0.255
 deny ip 172.16.33.0 0.0.0.255 172.16.10.0 0.0.0.255
 ! 允许访问Internet及其他
 permit ip 172.16.0.0 0.0.255.255 any

! === 应用到宿舍VLAN接口（入方向） ===
interface VLAN 31
 ip access-group DORM_TO_SERVER in

interface VLAN 32
 ip access-group DORM_TO_SERVER in

interface VLAN 33
 ip access-group DORM_TO_SERVER in

end
write memory
```

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

! 查看DHCP地址池状态
show ip dhcp pool

! 查看DHCP地址分配记录
show ip dhcp binding

! 查看DHCP服务器统计
show ip dhcp server statistics

! 查看ACL内容
show ip access-list

! 查看端口VLAN归属
show interfaces switchport

! 保存配置
write memory
```

---

## 预期路由表

```
Gateway of last resort is 172.16.100.2 to network 0.0.0.0

C    172.16.10.0/24 is directly connected, VLAN10
C    172.16.31.0/24 is directly connected, VLAN31
C    172.16.32.0/24 is directly connected, VLAN32
C    172.16.33.0/24 is directly connected, VLAN33
S    172.16.11.0/24 [1/0] via 172.16.101.2
S    172.16.21.0/24 [1/0] via 172.16.101.2
S    172.16.22.0/24 [1/0] via 172.16.101.2
C    172.16.100.0/30 is directly connected, FastEthernet0/24
C    172.16.101.0/30 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 172.16.100.2
```
