# 新路由3 刷机 + 校园网PJ + 访问内网 完整指南

> 适用于 SZTU（深圳技术大学）校园网环境，其他学校请根据实际情况修改参数。

---

## 一、准备工作

- 新路由3（Newifi3 D2）一台
- 网线两条
- 固件：`2021.1.24-LEDE-N3-V2.9.bin`（已编译好，支持科学上网、多拨、广告屏蔽）
- PuTTY 或任意 SSH 客户端

---

## 二、刷机后首次配置

电脑插网线连路由器 LAN 口，墙上校园网线插路由器 WAN 口。浏览器进入 `192.168.1.1` 路由器后台。

### 2.1 三件套脚本（SSH 进 192.168.1.1 后执行）

```bash
# 1. 开启 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. 开启 NAT 伪装（让所有设备共享一个 WAN IP 上网）
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE

# 3. 设置 TTL 为 128，伪装成单设备（防校园网多设备检测）
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128

# 4. 写入防火墙自启文件，防止重启丢失（用 heredoc，不要用 echo -e！）
cat > /etc/firewall.user << 'EOF'
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
EOF

chmod +x /etc/firewall.user
/etc/init.d/firewall restart
```

> ⚠️ **为什么不用 `echo -e`？** OpenWRT 的 BusyBox 默认不支持 `echo -e`，会把 `\n` 原样写进文件变成乱码，导致防火墙规则持久化失败、重启后丢失。**heredoc（`cat << 'EOF'`）是正确做法。**

### 2.2 验证

```bash
# 规则检查
iptables -t nat -L POSTROUTING -n --line-numbers    # 应有 1 条 MASQUERADE
iptables -t mangle -L POSTROUTING -n | grep TTL     # 应有 1 条 TTL set to 128
cat /proc/sys/net/ipv4/ip_forward                    # 应显示 1

# 连通性
ping -c 4 172.19.0.5    # 深澜认证页
ping -c 4 www.baidu.com # 外网
```

---

## 三、校园网多设备检测原理 & 破解原理

```
学校怎么发现你接了路由器？
  正常直连 → TTL=128
  经过路由器 NAT → 多一跳 → TTL=127
  学校看到同一个 IP 下 TTL 值不一致 → 判定多设备共享 → 封禁

破解：
  iptables mangle 表 → 所有出站包 TTL 统一设为 128
  学校看到：所有流量 TTL 都是 128，来自同一个 IP → 以为是同一台设备
```

| 组件 | 作用 | 缺了会怎样 |
|------|------|-----------|
| IP 转发 | 内核允许数据包在 LAN↔WAN 之间转发 | 内网设备完全无法访问外网 |
| NAT 伪装 (MASQUERADE) | 把所有设备流量换成 WAN 口 IP 发出 | 校园网不认识 192.168.1.x，直接丢包 |
| TTL 设为 128 | 统一 TTL，掩盖 NAT 多一跳 | 学校检测到 TTL 不一致，封 IP |

---

## 四、访问学校内网（教务系统、统一认证、SSDF 等）

### 4.1 核心问题

校园网的内网域名（如 `jwxt.sztu.edu.cn`）由学校自己的 DNS 服务器解析，**公网 DNS 查不到**。

### 4.2 学校 DNS 信息

| IP | 角色 | 备注 |
|----|------|------|
| `10.1.20.30` | DNS 服务器 | 解析 `*.sztu.edu.cn` |
| `10.1.20.92` | **网站 IP**（ssdf.sztu.edu.cn） | 这不是 DNS！之前搞错了 |
| `172.19.0.5` | 深澜认证 Web 页面 | 登录校园网账号用 |

### 4.3 已验证的学校内网域名与 IP 对照表

> 获取方法：**电脑直连校园网** → 终端执行 `nslookup 域名` → 记下返回的 IPv4 地址。

| 域名 | IP | 用途 |
|------|-----|------|
| `www.sztu.edu.cn` | `10.11.154.80` | 学校官网 |
| `jwxt.sztu.edu.cn` | `10.11.148.105` | 教务系统 |
| `ssdf.sztu.edu.cn` | `10.1.20.92` | 电费查询 |
| `auth.sztu.edu.cn` | `10.1.20.97` | 统一认证 |

### 4.4 添加新域名（每次遇到打不开的学校内网就做一次）

**步骤：**

1. 电脑**断开路由器 WiFi，网线直连校园网**
2. 终端执行：
```powershell
nslookup 你想查的域名.sztu.edu.cn
```
   记下返回的 IPv4 地址（如 `10.x.x.x`）
3. 重新连回路由器，SSH 进去执行：
```bash
echo "查到的IP[空格]域名" >> /etc/hosts
/etc/init.d/dnsmasq restart
```
4. 浏览器访问该域名验证

**一键批量添加（把上面表里的全部写入）：**
```bash
echo "10.11.154.80 www.sztu.edu.cn" >> /etc/hosts
echo "10.11.148.105 jwxt.sztu.edu.cn" >> /etc/hosts
echo "10.1.20.92 ssdf.sztu.edu.cn" >> /etc/hosts
echo "10.1.20.97 auth.sztu.edu.cn" >> /etc/hosts
/etc/init.d/dnsmasq restart
```

### 4.5 为什么用 hosts 而不是 dnsmasq 转发？

学校 DNS 服务器（`10.1.20.30`）能 ping 通但**不响应路由器的 DNS 查询**——它可能只答复已完成深澜认证的设备。dnsmasq 转发会超时，而 `hosts` 文件直接返回 IP，不依赖外部 DNS 查询。

---

## 五、常见故障排查

### 5.1 路由器重启后上不了网

SSH 进去重新执行 iptables 三件套：

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
/etc/init.d/firewall restart
```


### 5.2 路由器换地方（如宿舍→实验室）后上不了网

原因：WAN 口之前设了静态 IP，新地方的网段不一样。

解决：
1. 路由器后台把 WAN 口从"静态 IP"改为"DHCP 客户端"
2. SSH 进去手动执行 iptables 三件套
3. 重启防火墙

### 5.3 某个学校内网打不开

- `ping` 一下域名，能 ping 通但浏览器打不开 → DNS 正常，可能是网站自己的问题
- `ping` 不通 → 域名没写进 hosts，按第四章 4.4 步骤添加

---

## 六、可搭配的扩展玩法

| 玩法 | 说明 |
|------|------|
| 老路由器当 AP 扩展 WiFi | 老路由 LAN 口接新路由3 LAN 口，关 DHCP，WiFi 名密码设一样 |
| 树莓派 4B 跑 Docker | AdGuard Home（全屋去广告）、qBittorrent（PT 下载）、AList（网盘挂载）、Vaultwarden（自建密码管理器） |

---

## 七、验证清单

```bash
# 跑一遍确认一切正常

echo "===== 1. IP 转发 ====="
cat /proc/sys/net/ipv4/ip_forward

echo "===== 2. NAT 规则 ====="
iptables -t nat -L POSTROUTING -n | grep MASQUERADE

echo "===== 3. TTL 规则 ====="
iptables -t mangle -L POSTROUTING -n | grep TTL

echo "===== 4. 内网域名解析 ====="
nslookup www.sztu.edu.cn
nslookup jwxt.sztu.edu.cn
nslookup ssdf.sztu.edu.cn
nslookup auth.sztu.edu.cn

echo "===== 5. 连通性 ====="
ping -c 2 172.19.0.5
ping -c 2 www.baidu.com
ping -c 2 jwxt.sztu.edu.cn
```

预期：
- IP 转发：`1`
- NAT 规则：至少 1 条 MASQUERADE
- TTL 规则：1 条 `TTL set to 128`
- 四个域名都能解析出 `10.x.x.x` 内网 IP
- 全部 ping 通

---

整理 By：Mika + Claude
最后更新：2026.6.6
