# AWS EC2 实例多网卡（ENI）多公网 IP 配置文档

## 背景

实例 `i-06d5379fbcda741f3`（t3.xlarge）挂载了两块网卡，各自绑定一个弹性 IP（EIP），需要让两块网卡都能正常对外提供服务。

| 网卡 | 私有 IP | 弹性 IP (EIP) | 用途 |
|---|---|---|---|
| ens5（主网卡） | 172.31.31.47/20 | 18.61.117.97 | 主网卡，AWS 默认自动配置 |
| ens6（第二网卡） | 172.31.25.22/20 | 16.113.34.74 | 需要手动配置才能生效 |

子网：`172.31.16.0/20`，网关：`172.31.16.1`

---

## 一、问题现象与原因

### 1. 现象
- `ens6` 网卡状态为 `DOWN`，未分配 IP，导致公网 IP `16.113.34.74` 完全 ping 不通
- 即使手动激活网卡分配 IP 后，`ping 16.113.34.74` 依然 100% 丢包
- `ens5` 的 MTU 是 9001（Jumbo Frame），`ens6` 默认只有 1500

### 2. 根本原因

**AWS 控制台层面的配置（挂载 ENI、关联 EIP）和操作系统内部的网络配置是两回事**，AWS 只负责把网卡"接上"，网卡本身的 IP、启用状态、路由都需要 OS 自己配置。

**核心症结是"非对称路由"（Asymmetric Routing）**：
- Linux 默认只有一张主路由表，所有出方向流量都走**唯一的默认路由**（即 ens5）
- 当外部请求打到 `16.113.34.74`（ens6 的 EIP）时，请求包能正常到达实例
- 但系统在回复时，没有意识到"这个包该从 ens6 送出去、且要用 ens6 的私网 IP 作为源地址"，而是走了默认路由 ens5
- 结果：从 ens6 收到的请求，用 ens5 的身份回复，源地址和实际出口网卡不匹配
- **AWS 网络层有反欺骗（anti-spoofing）检查**，会直接丢弃这种源地址不匹配的包

解决方式：为 ens6 配置**基于源地址的策略路由（source-based policy routing）**，确保从 172.31.25.22 发出的包，强制走 ens6 和网关 172.31.16.1 出去。

---

## 二、（可选）手动调试阶段：创建命名路由表

在正式写永久配置之前，如果想用手动命令先验证思路是否可行，需要先创建一张独立的路由表：

```bash
# 给路由表编号 200 起一个便于识别的名字
echo "200 ens6-table" >> /etc/iproute2/rt_tables

# 本地子网路由 + 默认路由都写进这张表，走 ens6
ip route add 172.31.16.0/20 dev ens6 src 172.31.25.22 table ens6-table
ip route add default via 172.31.16.1 dev ens6 table ens6-table

# 核心规则：源地址是 172.31.25.22 的包，查 ens6-table
ip rule add from 172.31.25.22 lookup ens6-table

# 验证
ip rule show
ip route show table ens6-table
```

> 📌 **说明**：`/etc/iproute2/rt_tables` 这个文件的作用只是给数字表编号起个别名，纯粹是为了让 `ip route show table ens6-table` 这种命令更好读、便于人工排查。这一步只在**手动用 `ip` 命令临时调试**时需要。

> ⚠️ **注意**：这里创建的路由表和规则是**临时的**，重启后会丢失，仅用于验证思路是否可行。验证通过后，请按下一节写入 netplan 永久配置。

---

## 三、永久配置：netplan

Ubuntu 使用 netplan 管理网络配置，配置文件位于 `/etc/netplan/50-cloud-init.yaml`。

### 完整配置内容

> ⚠️ 请先用 `cat /etc/netplan/50-cloud-init.yaml` 确认你实际的 ens5 配置段（match/macaddress/set-name），下面以本次实际环境为例：

```yaml
network:
  ethernets:
    ens5:
      dhcp4: true
      dhcp6: false
      match:
        macaddress: 0e:db:8d:d0:c2:f5
      set-name: ens5
    ens6:
      dhcp4: true
      mtu: 9001
      dhcp4-overrides:
        route-metric: 200        # 比 ens5 大，避免抢占主默认路由
      routing-policy:
        - from: 172.31.25.22     # ens6 的私网 IP
          table: 200
      routes:
        - to: 0.0.0.0/0
          via: 172.31.16.1       # 子网网关
          table: 200
  version: 2
```

### 配置项说明

| 字段 | 作用 |
|---|---|
| `dhcp4: true` | 让 ens6 也通过 DHCP 自动获取私网 IP（AWS 会分配正确的 172.31.25.22） |
| `mtu: 9001` | 保持与 ens5 一致的 Jumbo Frame，VPC 内网通信效率更高 |
| `dhcp4-overrides.route-metric: 200` | 让 ens6 的默认路由优先级低于 ens5（ens5 是 100），避免两条默认路由冲突抢主 |
| `routing-policy` | **核心**：凡是源地址是 172.31.25.22 的包，查找路由表 200，而不是主路由表 |
| `routes`（table: 200） | 路由表 200 里定义：所有流量（0.0.0.0/0）都通过网关 172.31.16.1、从 ens6 出去 |

> 📌 netplan 里直接用数字表编号 `200`，不依赖 `/etc/iproute2/rt_tables` 里的命名（那是给第二节手动调试用的），两者是同一个编号，互不冲突。

---

## 四、应用配置（安全流程，避免 SSH 断连）

### 步骤 1：写入配置文件

**强烈建议用 heredoc 整体覆盖写入**，避免手动编辑产生缩进不一致导致的 YAML 语法错误（这是最常见的翻车点）：

```bash
cat > /etc/netplan/50-cloud-init.yaml << 'EOF'
network:
  ethernets:
    ens5:
      dhcp4: true
      dhcp6: false
      match:
        macaddress: 0e:db:8d:d0:c2:f5
      set-name: ens5
    ens6:
      dhcp4: true
      mtu: 9001
      dhcp4-overrides:
        route-metric: 200
      routing-policy:
        - from: 172.31.25.22
          table: 200
      routes:
        - to: 0.0.0.0/0
          via: 172.31.16.1
          table: 200
  version: 2
EOF
```

### 步骤 2：检查语法

```bash
# 确认没有 tab 字符混入（YAML 不允许 tab 缩进）
cat -A /etc/netplan/50-cloud-init.yaml | grep -n "	"

# 语法预检查
netplan generate
```

两条命令都没有报错才能继续下一步。

### 步骤 3：用 netplan try 安全应用

**不要直接用 `netplan apply`**，因为如果配置有误导致 ens5（主网卡/SSH 通道）出问题，可能直接把自己锁在外面。

```bash
sudo netplan try
```

- 该命令应用配置后会等待 **120 秒**
- 如果你不按回车确认，会**自动回滚**到之前的配置，避免彻底断连
- 应用后**另开一个终端窗口**，重新测试 SSH 连接和网络：

```bash
# 从实例外部（比如跳板机）测试两个 EIP 是否都通
ping 18.61.117.97
ping 16.113.34.74
```

- 确认都正常后，回到 `netplan try` 的窗口按 **Enter** 确认生效
- 如果连不上，等待 120 秒它会自动恢复原状，再排查配置问题

### 步骤 4：验证最终状态

```bash
# 检查网卡状态、IP、MTU
ip addr

# 检查策略路由规则
ip rule show

# 检查 table 200 的路由
ip route show table 200
```

预期看到：
```
ens6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 ...
    inet 172.31.25.22/20 ...
```

```
# ip rule show 应包含：
xxx: from 172.31.25.22 lookup 200
```

---

## 五、后续新增网卡时的操作要点（备忘）

以后如果再挂载第三块、第四块 ENI，重复以下要点即可：

1. 在 AWS 控制台创建 ENI 并挂载到实例，关联 EIP
2. 在 netplan 中为新网卡增加一段配置：`dhcp4: true` + 独立的 `route-metric`（依次递增，如 300、400）+ 对应的 `routing-policy` 和 `routes`（`table` 编号也要不同，如 300、400，避免和已有的冲突）
3. 用 `netplan generate` 检查语法 → `netplan try` 安全应用 → 外部验证 → 确认
4. 记得检查新网卡的 MTU 是否需要与主网卡保持一致

---

## 六、常见排错命令速查

| 命令 | 用途 |
|---|---|
| `ip addr` | 查看所有网卡的 IP、MTU、UP/DOWN 状态 |
| `ip route` | 查看主路由表 |
| `ip route show table 200` | 查看 ens6 专用路由表 |
| `ip rule show` | 查看策略路由规则是否生效 |
| `echo "200 ens6-table" >> /etc/iproute2/rt_tables` | （手动调试用）给数字路由表 200 起别名，方便查看 |
| `netplan generate` | 只检查语法，不实际应用 |
| `netplan try` | 安全应用配置，120 秒无确认自动回滚 |
| `cat -A file.yaml \| grep "	"` | 检查文件中是否混入 tab 字符 |
