# 分阶段验证测试指南

> 本指南按照"先连通、后安全、逐步叠加"的验证原则，提供完整的测试用例与验证命令。  
> 每个阶段前置条件：前一阶段测试全部通过。

---

## 验证流程总览

```
阶段0: 基础网络连通性测试
    ↓ 全部通过
阶段1: DHCP服务部署验证
    ↓ 全部通过
阶段2: DHCP Snooping部署验证
    ↓ 全部通过
阶段3: 802.1x认证部署验证
    ↓ 全部通过
阶段4: NAT部署验证
    ↓ 全部通过
阶段5: VPN部署验证
    ↓ 全部通过
项目交付
```

---

## 阶段0：基础网络连通性测试

> **前置条件**：已完成步骤1（基础网络与VLAN配置），尚未部署DHCP、802.1x、NAT、VPN等功能。  
> **测试环境**：所有终端PC配置静态IP地址，未启用任何安全功能。

### 0.1 VLAN内二层连通性测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| VLAN 11内通信 | A楼办公区PC1(172.16.11.10) ping PC2(172.16.11.11) | 互通 | `ping 172.16.11.11` |
| VLAN 21内通信 | B1教学楼PC1(172.16.21.10) ping PC2(172.16.21.11) | 互通 | `ping 172.16.21.11` |
| VLAN 31内通信 | C1宿舍PC1(172.16.31.10) ping PC2(172.16.31.11) | 互通 | `ping 172.16.31.11` |

**交换机验证命令：**

```shell
show vlan brief                    # 确认VLAN创建正确
show interfaces trunk              # 确认Trunk链路状态
show interfaces switchport         # 确认端口VLAN归属
```

### 0.2 跨VLAN三层路由测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 办公区→教学区 | A楼PC(172.16.11.10) ping B1楼PC(172.16.21.10) | 互通 | `ping 172.16.21.10` |
| 办公区→宿舍区 | A楼PC(172.16.11.10) ping C1楼PC(172.16.31.10) | 互通 | `ping 172.16.31.10` |
| 教学区→服务器区 | B1楼PC(172.16.21.10) ping服务器(172.16.10.10) | 互通 | `ping 172.16.10.10` |
| 宿舍区→服务器区 | C1楼PC(172.16.31.10) ping A楼PC(172.16.10.10) | 互通 | `ping 172.16.10.10` |

**核心交换机验证命令：**

```shell
show ip route                      # 确认路由表完整
show ip interface brief            # 确认SVI接口状态
ping 172.16.11.1 source 172.16.31.1  # 验证跨VLAN路由
```

### 0.3 路由表完整性验证

| 验证点 | 预期结果 | 验证命令 |
|--------|---------|---------|
| 核心交换机路由表 | 包含到办公区/教学区的静态路由，缺省路由指向出口 | `show ip route` |
| 汇聚交换机路由表 | 缺省路由指向核心 | `show ip route` |
| 出口路由器路由表 | 包含到内网的汇总路由，缺省路由指向ISP | `show ip route` |

**核心交换机预期路由表：**

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

---

## 阶段1：DHCP服务部署验证

> **前置条件**：阶段0测试通过，基础网络连通性正常。

### 1.1 DHCP地址获取测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| A楼办公区 | PC从DHCP获取地址 | 获取172.16.11.0/24网段地址 | `ipconfig /renew` |
| B1教学楼 | PC从DHCP获取地址 | 获取172.16.21.0/24网段地址 | `ipconfig /renew` |
| C1宿舍 | PC从DHCP获取地址 | 获取172.16.31.0/24网段地址 | `ipconfig /renew` |
| 网关正确性 | 获取的网关与VLAN对应 | 验证默认网关 | `ipconfig` 查看 Gateway |
| DNS正确性 | 获取的DNS为172.16.10.10 | 验证DNS服务器 | `ipconfig` 查看 DNS |

### 1.2 DHCP Relay功能验证

| 测试项 | 测试内容 | 预期结果 | 验证方法 |
|--------|---------|---------|---------|
| Relay工作正常 | 汇聚层正确转发DHCP请求 | 办公/教学区能获取地址 | 核心交换机 `show ip dhcp server statistics` |
| 跨网段分配 | 不同VLAN获取对应网段地址 | 地址与VLAN匹配 | 对比PC获取的地址与VLAN规划 |

**核心交换机验证命令：**

```shell
show ip dhcp pool                  # 查看地址池状态
show ip dhcp binding               # 查看地址分配记录
show ip dhcp server statistics     # 查看DHCP服务器统计
```

---

## 阶段2：DHCP Snooping部署验证

> **前置条件**：阶段1测试通过，DHCP服务正常工作。  
> **部署范围**：仅配置C楼接入交换机S2328G的DHCP Snooping（暂不启用802.1x）。

### 2.1 正常DHCP流程验证（Snooping启用后）

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 正常获取地址 | PC重新获取DHCP地址 | 仍能正常获取地址 | `ipconfig /release` 后 `ipconfig /renew` |
| 地址与之前一致 | 获取的地址与阶段1相同 | 地址分配正常 | 对比IP地址 |

### 2.2 非法DHCP服务器阻断测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 私接路由器测试 | 在宿舍区接入层私接路由器（开启DHCP） | 其他PC无法从该路由器获取地址 | 观察PC是否获取到192.168.x.x地址 |
| Offer报文拦截 | 非法DHCP Offer被丢弃 | PC仅获取到合法地址 | `show ip dhcp snooping` 查看统计 |

**C楼接入交换机验证命令：**

```shell
show ip dhcp snooping              # 查看Snooping状态
show ip dhcp snooping binding      # 查看绑定表
show interfaces status             # 查看Trusted/Untrusted端口状态
```

### 2.3 DHCP Snooping绑定表验证（防止私设IP）

| 测试项 | 测试内容 | 预期结果 | 验证方法 |
|--------|---------|---------|---------|
| 绑定表生成 | 正常DHCP用户被记录到绑定表 | 表中有MAC-IP-端口-VLAN映射 | `show ip dhcp snooping binding` |
| 绑定表正确性 | 绑定表信息与实际情况一致 | MAC、IP、端口、VLAN匹配 | 对比PC实际信息 |
| 私设IP拦截 | 手动配置静态IP，尝试访问网关或其他网段 | 数据包被丢弃，无法通信 | `show ip dhcp snooping statistics` 查看 match failures 计数增加 |

**预期绑定表示例：**

```
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  -----------
00:11:22:33:44:55   172.16.31.10     14400       dhcp-snooping  31    FastEthernet0/2
00:11:22:33:44:66   172.16.31.11     14400       dhcp-snooping  31    FastEthernet0/3
```

### 2.4 源MAC校验功能测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| CHADDR伪造攻击 | 使用工具伪造CHADDR发送DHCP Discover | 报文被丢弃，无法耗尽地址池 | 观察地址池剩余地址数量 |
| 校验统计 | 伪造报文被计数 | `show ip dhcp snooping` 显示丢弃计数增加 | 查看交换机统计 |

---

## 阶段3：802.1x认证部署验证

> **前置条件**：阶段2测试通过，DHCP Snooping正常工作。  
> **部署范围**：C楼接入交换机802.1x配置 + RADIUS服务器配置。

### 3.1 未认证状态隔离测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 端口初始状态 | 未认证端口状态为Unauthorized | 仅允许EAPOL报文通过 | `show dot1x interface fa0/2` |
| 网络隔离 | 未认证PC无法ping通任何地址 | 所有IP流量被阻断 | `ping 172.16.31.1` 失败 |

**C楼接入交换机验证命令：**

```shell
show dot1x                         # 查看802.1x全局状态
show dot1x interface fa0/2         # 查看指定端口认证状态
show dot1x interface fa0/2 statistics  # 查看认证统计
```

**预期未认证状态：**

```
show dot1x interface fa0/2
  PortControl = AUTO
  ControlDirection = Both
  HostMode = SINGLE_HOST
  Reauthentication = Enabled
  QuietPeriod = 60 seconds
  Status = UNAUTHORIZED          <-- 未授权状态
```

### 3.2 认证流程测试（合法用户）

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| EAPOL交互 | 终端发起802.1x认证 | EAPOL报文正常交换 | Wireshark抓包查看EAP包 |
| RADIUS通信 | 交换机与RADIUS服务器通信正常 | Access-Accept返回 | RADIUS服务器日志查看 |
| 认证后状态 | 端口状态变为Authorized | 状态为authenticated | `show dot1x interface fa0/2` |
| 认证后网络 | 认证后PC能够通信 | 正常通信 | ping网关或其他 |

**预期认证成功状态：**

```
show dot1x interface fa0/2
  Status = AUTHORIZED            <-- 已授权状态
```

### 3.3 非法用户认证失败测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 错误密码 | 输入错误密码认证 | 认证失败，端口保持Unauthorized | 观察认证提示 |
| 不存在账号 | 使用不存在的账号 | 认证失败 | 观察认证提示 |
| 失败后隔离 | 认证失败后PC仍被隔离 | 无法通信 | ping失败 |

### 3.4 单主机模式测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 首台设备认证 | 第一台PC认证成功 | 正常访问网络 | 完成认证流程 |
| MAC修改测试 | 在PC1上手动修改网卡MAC为另一个地址 | 无法认证或导致端口err-disabled | 观察端口状态 |

**验证命令：**

```shell
show dot1x interface fa0/2         # 查看端口状态和主机数
show interfaces status err-disabled  # 查看是否进入err-disabled状态
```

---

## 阶段4：NAT部署验证

> **前置条件**：阶段3测试通过，802.1x认证正常工作。  
> **部署范围**：出口路由器NAPT配置。

### 4.1 内网访问Internet测试

> **提示**：ICMP老化时间短，使用 `-t` 参数持续ping来观察。

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 办公区访问公网 | A楼PC ping 111.100.0.6 | 通 | `ping 111.100.0.6 -t` |
| 教学区访问公网 | B1楼PC ping 111.100.0.6 | 通 | `ping 111.100.0.6 -t` |
| 宿舍区访问公网 | C1楼认证后PC ping 111.100.0.6 | 通 | `ping 111.100.0.6 -t` |

### 4.2 NAT转换表验证

| 测试项 | 测试内容 | 预期结果 | 验证方法 |
|--------|---------|---------|---------|
| 转换表生成 | 内网访问公网后生成转换条目 | 表中有Inside/Outside映射 | `show ip nat translations` |
| 端口复用 | 多内网主机共用同一公网地址 | 不同高端口映射不同内网IP | 查看转换表端口分配 |

**出口路由器验证命令：**

```shell
show ip nat translations           # 查看NAT转换表
show ip nat statistics             # 查看NAT统计
show ip nat translations verbose   # 查看详细转换信息
```

**预期转换表示例：**

```
Pro  Inside global      Inside local        Outside local       Outside global
---  111.100.0.5:1025   172.16.11.10:5000   111.100.0.6:53      111.100.0.6:53
---  111.100.0.5:1026   172.16.21.10:5001   111.100.0.6:53      111.100.0.6:53
---  111.100.0.5:1027   172.16.31.10:5002   111.100.0.6:53      111.100.0.6:53
```

---

## 阶段5：VPN部署验证

> **前置条件**：阶段4测试通过，NAT正常工作。  
> **部署范围**：出口路由器IPsec VPN + PPTP VPN，家属区路由器IPsec VPN。

### 5.1 IPsec VPN隧道建立测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| IKE SA建立 | 触发流量后IKE协商成功 | ISAKMP SA状态为QM_IDLE | `show crypto isakmp sa` |
| IPsec SA建立 | IKE成功后建立IPsec SA | 有双向SA条目 | `show crypto ipsec sa` |
| 隧道状态 | 隧道状态为Active | 状态正常 | `show crypto map` |

**出口路由器验证命令：**

```shell
show crypto isakmp sa              # 查看IKE SA
show crypto ipsec sa               # 查看IPsec SA
show crypto map                    # 查看Crypto Map状态
```

**预期IKE SA状态：**

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id slot status
111.100.0.5     202.115.21.2    QM_IDLE           1001    0 ACTIVE
```

### 5.2 IPsec VPN通信测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| 家属区→校园网 | 家属区PC(192.168.1.10) ping校园网服务器(172.16.10.10) | 通，流量经IPsec隧道加密 | `ping 172.16.10.10` |
| 校园网→家属区 | 校园网PC ping家属区PC | 通 | `ping 192.168.1.10` |
| 加密验证 | 抓包验证ESP封装 | 看到ESP协议（协议号50）或UDP 4500 | Wireshark抓包 |

### 5.3 PPTP VPN连接测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| PPTP连接建立 | 移动用户发起PPTP连接 | 连接成功，获取10.200.200.x地址 | Windows VPN客户端 |
| 地址分配 | 获取的地址在地址池范围内 | 获取10.200.200.0/24地址 | `ipconfig` 查看 |
| 会话状态 | 路由器显示PPTP会话 | 有active会话 | `show vpdn session` |

**出口路由器验证命令：**

```shell
show vpdn                          # 查看VPDN状态
show vpdn session                  # 查看PPTP会话
show vpdn tunnel                   # 查看PPTP隧道
show users                         # 查看登录用户
```

### 5.4 PPTP VPN通信测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| VPN用户→内网 | PPTP用户ping校园网服务器 | 通 | `ping 172.16.10.10` |
| VPN用户→办公区 | PPTP用户ping办公区PC | 通 | `ping 172.16.11.10` |
| 回程路由 | 内网服务器能回应VPN用户 | 通 | 服务器ping VPN用户 |

### 5.5 NAT-Traversal功能测试

| 测试项 | 测试内容 | 预期结果 | 测试方法 |
|--------|---------|---------|---------|
| NAT-T启用 | ESP封装在UDP 4500 | 抓包看到UDP 4500 | Wireshark抓包 |
| PPTP GRE穿越 | GRE协议正确穿越NAT | PPTP数据通道正常 | 验证PPTP通信 |

---

## 验证问题速查表

| 故障现象 | 可能原因 | 排查步骤 |
|---------|---------|---------|
| VLAN内不通 | 端口未加入VLAN / Trunk不通 | `show vlan brief` + `show interfaces trunk` |
| 跨VLAN不通 | 路由缺失 / SVI未up | `show ip route` + `show ip interface brief` |
| DHCP获取失败 | Relay未配置 / 地址池耗尽 | `show ip dhcp pool` + `show ip dhcp server statistics` |
| Snooping阻断合法DHCP | Trusted端口配置错误 | `show ip dhcp snooping` |
| 802.1x无响应 | RADIUS不可达 / 端口未启用 | `ping 172.16.10.10` + `show dot1x` |
| NAT不转换 | ACL未命中 / 接口标识错误 | `show ip nat translations` + `show access-list` |
| IPsec隧道不通 | IKE策略不匹配 / 感兴趣流错误 | `show crypto isakmp sa` + `show access-list 101` |
| PPTP无法拨入 | VPDN未启用 / 地址池冲突 | `show vpdn` + `show ip local pool` |
