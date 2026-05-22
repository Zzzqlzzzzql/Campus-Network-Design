# 接入交换机配置 — C楼宿舍区 S2328G

> 设备型号：锐捷 S2328G（二层交换机）  
> 部署位置：C楼学生宿舍区  
> 功能定位：宿舍用户接入、802.1x端口认证、DHCP Snooping安全防护  
> 主机名：`Access-C-S2328`

---

## 接口分配

| 接口 | 类型 | VLAN | 说明 |
|------|------|------|------|
| Fa0/1 | Trunk | 31,32,33,40 | 上联核心交换机 S3760-24（Trusted） |
| Fa0/2-8 | Access | 31 | C1宿舍用户端口 |
| Fa0/9-16 | Access | 32 | C2宿舍用户端口 |
| Fa0/17-24 | Access | 33 | C3宿舍用户端口 |

---

## 配置命令

### 1. 基础配置与VLAN创建

```shell
enable
configure terminal
hostname Access-C-S2328

! === 创建宿舍VLAN ===
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

### 2. 管理配置与上联接口

```shell
! === 管理VLAN接口（仅用于管理和RADIUS通信） ===
interface VLAN 40
 ip address 172.16.40.2 255.255.255.252
 no shutdown
 description Mgmt_Interface

! === 默认网关指向核心 ===
ip default-gateway 172.16.40.1

! === 上联核心的Trunk ===
interface FastEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 31,32,33,40
 no shutdown
 description To_Core_Direct
```

### 3. 用户端口基础配置

```shell
! === C1宿舍端口（Fa0/2-8，VLAN 31） ===
interface range FastEthernet 0/2-8
 switchport mode access
 switchport access vlan 31
 spanning-tree portfast
 description Dorm_C1_Ports

! === C2宿舍端口（Fa0/9-16，VLAN 32） ===
interface range FastEthernet 0/9-16
 switchport mode access
 switchport access vlan 32
 spanning-tree portfast
 description Dorm_C2_Ports

! === C3宿舍端口（Fa0/17-24，VLAN 33） ===
interface range FastEthernet 0/17-24
 switchport mode access
 switchport access vlan 33
 spanning-tree portfast
 description Dorm_C3_Ports
```

### 4. DHCP Snooping配置

```shell
! === DHCP Snooping全局启用 ===
! 【v5.3适配】锐捷命令：ip dhcp snooping enable
! （原Cisco命令：ip dhcp snooping）
ip dhcp snooping enable

! === 启用源MAC校验（防范CHADDR伪造攻击） ===
! 【v5.3适配】锐捷命令：ip dhcp snooping check source-mac-address
! （原Cisco命令：ip dhcp snooping verify mac-address）
ip dhcp snooping check source-mac-address

! === 上联端口设为Trusted ===
! 仅Trusted端口允许DHCP Offer/ACK/NAK报文通过
interface FastEthernet 0/1
 ip dhcp snooping trust
 description Uplink_to_Core_Trusted

! === 用户端口启用Snooping和IP Source Guard ===
! 【v5.3适配】锐捷利用IP Source Guard实现"仅允许DHCP分配地址通信"
! （原Cisco命令：ip dhcp snooping address-bind）
interface range FastEthernet 0/2-24
 ip verify source port-security
 ! ip dhcp snooping limit rate 5    ! 可选：DHCP报文速率限制
```

### 5. 802.1x认证配置

```shell
! === 802.1x全局启用 ===
! 【v5.3适配】锐捷命令：dot1x enable
! （原Cisco命令：dot1x system-auth-control）
dot1x enable

! === RADIUS服务器配置 ===
! 指向D楼服务器 172.16.10.10
radius-server host 172.16.10.10 auth-port 1812 acct-port 1813 key UESTC2026

! === AAA认证授权 ===
! 认证请求转发至RADIUS服务器
aaa authentication dot1x default group radius

! 【v5.3说明】以下命令为Cisco风格，锐捷S2328G老版本在静态VLAN场景下
! 通常无需配置authorization，若需RADIUS动态下发VLAN/ACL，
! 请确认具体RGOS子版本是否支持该命令。
! aaa authorization network default group radius
```

### 6. 端口802.1x启用

```shell
! === 在全部宿舍端口启用802.1x ===
interface range FastEthernet 0/2-24
 ! 【v5.3适配】锐捷必须在接口下单独 enable
 dot1x enable
 ! 端口控制模式：auto表示未认证时阻塞，认证后放行
 dot1x port-control auto
 ! 【v5.3删除】S2328G老版本RGOS不支持 host-mode 配置
 ! dot1x host-mode single-host
 ! 启用定期重认证
 dot1x re-authentication enable
 ! 【v5.3适配】锐捷命令：dot1x quiet-period 60
 ! （原Cisco命令：dot1x timeout quiet-period 60）
 dot1x quiet-period 60
 spanning-tree portfast

end
write memory
```

---

## 安全配置原理

### 802.1x 认证流程

```
1. 终端连接端口
   ↓
2. 端口初始状态 = UNAUTHORIZED
   仅允许 EAPOL 报文（目的MAC 01-80-C2-00-00-03）
   阻断所有 IP 业务数据
   ↓
3. 终端发起 EAPOL-Start
   交换机转发 EAP-Request/Identity 至终端
   终端回应 EAP-Response/Identity（含用户名）
   ↓
4. 交换机封装 RADIUS Access-Request 发送至 RADIUS Server (172.16.10.10)
   RADIUS Server 验证凭据
   ↓
5a. 认证成功 → RADIUS返回 Access-Accept
    端口状态切换为 AUTHORIZED
    开放对应 VLAN 的转发权限
    ↓
5b. 认证失败 → RADIUS返回 Access-Reject
    端口保持 UNAUTHORIZED
    进入 Quiet Period（60秒）后允许重试
```

### DHCP Snooping 信任边界

| 端口类型 | DHCP角色 | 行为 |
|---------|---------|------|
| **Trusted Port**（上联口 Fa0/1） | 允许DHCP服务器响应 | 允许Offer/ACK/NAK报文通过 |
| **Untrusted Port**（用户口 Fa0/2-24） | 仅允许DHCP客户端请求 | 丢弃服务器响应，阻断Rogue DHCP |

### 安全机制联动

```
802.1x认证（准入控制）
  + DHCP Snooping（地址安全）
  + IP Source Guard（源地址验证）
  = 三重防护：未认证隔离 + 非法DHCP阻断 + 私设IP拦截
```

---

## 验证命令

```shell
! 查看VLAN信息
show vlan brief

! 查看Trunk链路状态
show interfaces trunk

! === DHCP Snooping相关 ===
! 查看Snooping状态
show ip dhcp snooping
! 查看绑定表（IP-MAC-端口-VLAN映射）
show ip dhcp snooping binding
! 查看端口Trusted/Untrusted状态
show interfaces status

! === 802.1x相关 ===
! 查看802.1x全局状态
show dot1x
! 查看指定端口认证状态
show dot1x interface fa0/2
! 查看认证统计
show dot1x interface fa0/2 statistics
! 查看err-disabled端口
show interfaces status err-disabled

! === 管理平面 ===
! 验证与RADIUS服务器连通性
ping 172.16.10.10 source 172.16.40.2

! 保存配置
write memory
```

---

## 预期端口状态

### 未认证状态（Unauthorized）

```
show dot1x interface fa0/2
  PortControl = AUTO
  ControlDirection = Both
  HostMode = SINGLE_HOST
  Reauthentication = Enabled
  QuietPeriod = 60 seconds
  Status = UNAUTHORIZED          <-- 未授权，IP流量被阻断
```

### 认证成功状态（Authorized）

```
show dot1x interface fa0/2
  Status = AUTHORIZED            <-- 已授权，正常转发
```

---

## 常见问题排查

| 问题 | 可能原因 | 排查命令 |
|------|---------|---------|
| 终端无法获取IP | 802.1x未认证通过 | `show dot1x interface fa0/x` |
| 认证无响应 | RADIUS服务器不可达 | `ping 172.16.10.10` |
| 获取到非法IP(192.168.x.x) | DHCP Snooping未生效 | `show ip dhcp snooping` |
| 私设静态IP仍能通信 | IP Source Guard未启用 | `show ip verify source` |
| 端口持续err-disabled | MAC漂移或违规接入 | `show interfaces status err-disabled` |
