# 接入交换机配置 — AB楼 WS-C2912

> 设备型号：思科 WS-C2912（二层交换机）  
> 部署位置：A楼+B楼  
> 功能定位：办公区与教学区用户接入、端口VLAN划分  
> 主机名：默认（未指定，可选配置）

---

## 接口分配

| 接口 | 类型 | VLAN | 说明 |
|------|------|------|------|
| Fa0/1 | Trunk | 11,21,22 | 上联汇聚交换机 S3760-24 |
| Fa0/2-4 | Access | 11 | A楼办公区端口 |
| Fa0/5-8 | Access | 21 | B1教学楼端口 |
| Fa0/9-12 | Access | 22 | B2教学楼端口 |

---

## 配置命令

### 1. VLAN数据库配置

```bash
enable

! === VLAN创建（旧版Cisco使用vlan database模式）===
vlan database
vlan 11 name Office
vlan 21 name Teaching_B1
vlan 22 name Teaching_B2
exit
```

### 2. 端口配置

```bash
configure terminal

! === 上联汇聚层（Trunk） ===
interface FastEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 11,21,22
 switchport trunk encapsulation dot1q
 description To_Aggre_AB

! === A楼办公区（Fa0/2-4，VLAN 11） ===
interface range FastEthernet 0/2-4
 switchport mode access
 switchport access vlan 11
 spanning-tree portfast
 description Office_A_Ports

! === B1楼教学区（Fa0/5-8，VLAN 21） ===
interface range FastEthernet 0/5-8
 switchport mode access
 switchport access vlan 21
 spanning-tree portfast
 description Teaching_B1_Ports

! === B2楼教学区（Fa0/9-12，VLAN 22） ===
interface range FastEthernet 0/9-12
 switchport mode access
 switchport access vlan 22
 spanning-tree portfast
 description Teaching_B2_Ports

end
write memory
```

---

## 技术说明

### 设备特性说明

WS-C2912 为思科旧款二层交换机，配置时需注意以下特性：

| 特性 | 说明 |
|------|------|
| VLAN创建方式 | 使用 `vlan database` 模式（非 `vlan` 全局配置模式） |
| Trunk封装 | 需显式指定 `switchport trunk encapsulation dot1q` |
| 端口范围 | 支持 `interface range` 命令批量配置 |
| 生成树 | 默认启用STP，接入端口建议配置 `spanning-tree portfast` 加速收敛 |

### 端口隔离

办公区与教学区通过VLAN实现二层隔离：
- **VLAN 11（A楼办公区）**：独立广播域，120个信息点
- **VLAN 21（B1教学楼）**：独立广播域，100个信息点  
- **VLAN 22（B2教学楼）**：独立广播域，100个信息点

跨VLAN通信需经汇聚层S3760-24三层转发。

---

## 验证命令

```shell
! 查看VLAN信息
show vlan brief

! 查看Trunk链路状态
show interfaces trunk

! 查看端口VLAN归属
show interfaces switchport

! 查看生成树状态
show spanning-tree

! 查看接口状态
show interfaces status

! 保存配置
write memory
```
