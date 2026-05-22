# RADIUS/AAA服务器配置

> 部署位置：D楼网络中心 VLAN 10（172.16.10.0/24）  
> 系统平台：Windows Server + NPS（Network Policy Server）  
> 服务器IP：172.16.10.10/24，网关：172.16.10.1  
> 共享密钥：UESTC2026

---

## 1. 服务器基础配置

### 网络参数

| 参数 | 值 |
|------|------|
| IP地址 | 172.16.10.10 |
| 子网掩码 | 255.255.255.0 |
| 默认网关 | 172.16.10.1（核心交换机SVI 10） |
| DNS | 172.16.10.10（自身） / 8.8.8.8 |

### 服务器部署

1. 在D楼网络中心安装Windows Server（建议2016或更高版本）
2. 配置双网卡（如需要管理平面分离），至少一个网卡接入VLAN 10
3. 固定IP为 172.16.10.10/24
4. 确保与核心交换机Fa0/3-5接口物理连通

---

## 2. NPS角色安装

### 安装步骤

1. 打开 **服务器管理器** → **添加角色和功能**
2. 选择 **网络策略和访问服务 (Network Policy and Access Services)**
3. 勾选 **网络策略服务器 (Network Policy Server, NPS)**
4. 完成安装

### 服务确认

安装完成后，确认NPS服务已启动：

```powershell
# PowerShell 检查NPS服务状态
Get-Service ias | Select-Object Name, Status
# 预期输出：Status = Running
```

---

## 3. RADIUS客户端配置

### 添加接入交换机为RADIUS客户端

| 参数 | 值 |
|------|------|
| 客户端名称 | Dorm-Access-S2328G |
| IP地址 | 172.16.40.2（通过VLAN 40与RADIUS服务器通信） |
| 共享密钥 | UESTC2026 |
| 供应商名称 | RADIUS Standard（或选择Ruijie） |

### 配置步骤

1. 打开 **NPS管理控制台**（`nps.msc`）
2. 右键 **RADIUS客户端** → **新建**
3. 填写友好名称：`Dorm-Access-S2328G`
4. 填写IP地址：`172.16.40.2`
5. 填写共享密钥：`UESTC2026`（或选择生成并手动同步到交换机配置）
6. 确认配置

> **注意**：C楼接入交换机管理IP为172.16.40.2（通过VLAN 40与RADIUS服务器通信）。若实际实验中管理IP不同，请相应调整RADIUS客户端配置和交换机配置中的`radius-server host`地址。

---

## 4. 连接请求策略

### 策略配置

1. 在NPS控制台中，展开 **策略** → **连接请求策略**
2. 新建策略，名称：`802.1x-Dorm-Policy`
3. 条件：选择 **日期和时间限制**（或留空为始终允许）
4. 身份验证设置：
   - 选择 **在此服务器上处理请求**（本地处理，不转发至外部RADIUS）
5. 完成配置

---

## 5. 网络策略（认证策略）

### 策略配置

1. 在NPS控制台中，展开 **策略** → **网络策略**
2. 新建策略，名称：`Dorm-Auth-Policy`
3. 条件配置：
   - 添加 **Windows组** 或特定用户组（如 `Domain\DormUsers`）
   - 或添加 **RADIUS客户端属性** 限定来源客户端
4. 访问权限：选择 **授予访问权限**
5. EAP类型配置：
   - 选择 **Microsoft: 受保护的EAP (PEAP)**
   - 点击 **配置**，导入或创建服务器证书
   - 内部方法选择 **Secured password (EAP-MSCHAPv2)**
6. 约束设置（可选）：
   - 设置空闲超时、会话超时等
7. 设置：
   - RADIUS属性 → 标准 → 添加 **Filter-Id**（用于动态下发ACL，若交换机支持）
8. 完成配置

### EAP类型说明

| 协议 | 说明 | 适用场景 |
|------|------|---------|
| PEAP-MSCHAPv2 | 使用TLS隧道封装MSCHAPv2认证 | 推荐，安全性高，兼容性好 |
| EAP-TLS | 基于客户端证书的双向认证 | 最高安全性，需部署证书体系 |
| EAP-MD5 | 挑战握手认证（仅用于MAB） | 用于MAC Authentication Bypass |

---

## 6. 用户账户配置

### 创建测试用户

在Active Directory（或本地SAM）中创建测试用户：

```powershell
# PowerShell 创建AD用户（需要RSAT-AD-PowerShell）
New-ADUser `
  -Name "dormuser1" `
  -UserPrincipalName "dormuser1@domain.local" `
  -AccountPassword (ConvertTo-SecureString "UESTC@2026" -AsPlainText -Force) `
  -Enabled $true `
  -Path "OU=DormUsers,DC=domain,DC=local"

# 将用户加入认证组
Add-ADGroupMember -Identity "DormUsers" -Members "dormuser1"
```

或使用本地用户（无AD环境）：

```cmd
:: CMD 创建本地用户
net user dormuser1 UESTC@2026 /add
net localgroup "Network Policy Server Users" dormuser1 /add
```

---

## 7. 证书配置（PEAP必需）

### 服务器证书导入

PEAP-MSCHAPv2要求RADIUS服务器具有有效的服务器证书：

**方式A：域CA自动颁发（推荐）**
1. 部署企业CA（Active Directory证书服务）
2. 在NPS服务器上申请 **计算机** 证书模板
3. 确保证书存储在 **计算机\个人** 存储区

**方式B：自签名证书（测试环境）**
```powershell
# PowerShell 创建自签名证书（测试用途）
New-SelfSignedCertificate `
  -DnsName "radius.domain.local" `
  -CertStoreLocation "cert:\LocalMachine\My" `
  -KeyUsage DigitalSignature,KeyEncipherment `
  -Provider "Microsoft RSA SChannel Cryptographic Provider"
```

**方式C：购买商业证书（生产环境）**
- 向公共CA申请服务器身份验证证书
- 确保SAN（主题备用名称）包含RADIUS服务器FQDN

### 证书验证

```powershell
# 查看计算机个人证书存储
Get-ChildItem -Path "cert:\LocalMachine\My" | Where-Object { $_.Subject -like "*radius*" }

# 确认证书有效、未过期、私钥可访问
```

---

## 8. 记账与日志配置

### 启用日志记录

1. 在NPS控制台中，右键点击 **NPS(本地)** → **属性**
2. 切换到 **日志记录** 选项卡
3. 配置以下日志方式（建议同时启用）：

#### IAS格式日志（本地文件）

- 日志文件位置：`C:\Windows\System32\LogFiles\IAS\`
- 日志文件名：`INYYYYMMDD.log`
- 格式：符合IAS（Internet Authentication Service）标准格式

#### SQL Server日志（可选，生产推荐）

- 配置ODBC数据源连接到SQL Server
- 创建NPS记账数据库表

### 日志分析

日志记录以下关键事件：

| 事件ID | 说明 |
|--------|------|
| 6272 | 网络策略服务器授予用户访问权限 |
| 6273 | 网络策略服务器拒绝用户访问请求 |
| 6274 | 网络策略服务器丢弃用户访问请求 |
| 6278 | 网络策略服务器授予用户完整访问权限（符合策略） |

---

## 9. 验证测试

### 交换机侧验证RADIUS连通性

在C楼接入交换机上执行：

```shell
! 验证与RADIUS服务器三层连通
ping 172.16.10.10 source 172.16.40.2

! 查看RADIUS服务器状态
show radius server

! 查看802.1x认证统计
show dot1x statistics
```

### RADIUS服务器侧验证

1. 打开 **事件查看器** → **应用程序和服务日志** → **Microsoft\Windows\NetworkPolicyServer`
2. 查看操作日志，确认收到来自交换机（172.16.40.2）的Access-Request
3. 检查认证成功/失败事件

### 测试认证流程

1. 在宿舍区PC上启用802.1x认证（Windows：网络和共享中心 → 更改适配器设置 → 右键网卡 → 身份验证 → 启用IEEE 802.1x）
2. 输入测试账户：`dormuser1` / `UESTC@2026`
3. 在NPS日志中查看认证结果
4. 在交换机上验证端口状态变为 `AUTHORIZED`

---

## 10. 高可用建议（可选）

| 方案 | 说明 |
|------|------|
| NPS冗余 | 部署第二台NPS服务器，在交换机配置中指定主/备RADIUS服务器 |
| RADIUS代理 | 使用NPS作为RADIUS代理，将请求转发至外部RADIUS集群 |
| AD站点感知 | 确保NPS服务器与GC/DC在同一站点，减少认证延迟 |

### 交换机冗余配置

```shell
! 主RADIUS服务器
radius-server host 172.16.10.10 auth-port 1812 acct-port 1813 key UESTC2026
! 备RADIUS服务器（可选）
radius-server host 172.16.10.11 auth-port 1812 acct-port 1813 key UESTC2026
```
