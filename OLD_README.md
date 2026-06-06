# 已验证新路由3刷此固件并且进行脚本修改TTL可实现路由器上网
## 此固件为网上找到的别人编译好的，支持科学，多拨，广告屏蔽（我自己编译和云编译出来的固件总是出错）
> [!CAUTION]
># 准备两条网线，全程电脑插着网线连接路由器LAN口，然后墙上网线连接路由器WAN口在刷完固件之后，请先把路由器的WAN口（浏览器进入192.168.1.1路由器后台）和电脑设置”网络偏好设置”的“以太网”的“属性”设为静态ip，将参数和你提前先用电脑连接sztu校园网的参数一样，注意路由器WAN口设置的子网掩码需要根据校园网给你分配的ip地址进行计算，具体换算为
## 假如我的ip地址为x.x.x.x/23
![在路由器后台查看到的ip](screenshot1.png)

CIDR 前缀 /23 的含义
```text
/23 表示：子网掩码的二进制形式中，前 23 位 是 1，后面的位是 0。  
子网掩码是 32 位二进制数，分成 4 段（每段 8 位）：

/23 → 二进制：11111111.11111111.11111110.00000000

逐段把二进制转成十进制：

二进制段	十进制值	
11111111	255	
11111111	255	
11111110	254	
00000000	0	
所以，/23 对应的子网掩码就是：255.255.254.0 
```
```
你电脑以太网的网段是 192.168.1.0/24：
/24 → 二进制：11111111.11111111.11111111.00000000
转十进制：255.255.255.0
```
#### ~~其实我说问ai最快~~

## 电脑设置为静态ip效果如图（例）：
![在网络偏好设置看到的](screenshot2.png)
```
电脑以太网 IP 是 192.168.1.3，路由器 LAN 口是 192.168.1.1。
路由器 LAN 口的网段是 192.168.1.0/24，对应的子网掩码就是 255.255.255.0。
把电脑以太网子网掩码改成 255.255.255.0，电脑和路由器 LAN 口就完全在同一个网段里，访问 192.168.1.1 会更稳定，也不会和其他网段产生干扰。
```
### 刷机后使用PuTTY（或者其它软件） SSH连接192.168.1.1后粘贴以下脚本即可：
```bash

#开启IP转发
echo 1 > /proc/sys/net/ipv4/ip_forward
#开启NAT（伪装）
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
#设置TTL为128，伪装成单设备,我们学校是128，具体多少看你学校
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
#保存规则到防火墙自启（重启不丢）
echo -e "echo 1 > /proc/sys/net/ipv4/ip_forward\niptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE\niptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128" >> /etc/firewall.user
chmod +x /etc/firewall.user
/etc/init.d/firewall restart
```
## 执行完以上命令，路由器重启后可能会失效，然后再执行：
```bash
# 开启 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 开启 NAT 伪装（让其他设备能上网）
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE

# 设置 TTL 为 128，防多设备检测
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
#把规则写入 /etc/firewall.user，这样每次开机都会自动加载：
cat >> /etc/firewall.user << 'EOF'
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
EOF

chmod +x /etc/firewall.user
/etc/init.d/firewall restart
```
## 祝你成功，别忘了在LuCI界面设置无线网络的名称密码以及是否开启SSID广播
### 有问题请~~STFAI~~STFW
> [!TIP]
> ## 有时候修改完设置重启路由器依然上不了网，就再ssh连接192.168.1.1执行第二段命令就好了
>

# **2026.6.6更新：**
## 这一版脚本是claude总结的命令，似乎更简洁

```bash

uci set dhcp.@dnsmasq[0].server="/sztu.edu.cn/10.1.20.92"
uci commit dhcp

# 同时把学校 DNS 加入上游服务器列表（兜底）
echo "server=10.1.20.92" >> /etc/dnsmasq.conf

# 重启 dnsmasq 使 DNS 配置生效
/etc/init.d/dnsmasq restart

# ===== 第2步：让内网流量跳过 NAT 伪装，保持原始源 IP =====
# 先删掉原来的全局 MASQUERADE 规则
iptables -t nat -D POSTROUTING -o eth0.2 -j MASQUERADE 2>/dev/null

# 重新按次序加规则：内网地址段先放行（不做伪装），外网才伪装
iptables -t nat -A POSTROUTING -o eth0.2 -d 10.0.0.0/8     -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -d 172.16.0.0/12  -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -d 192.168.0.0/16 -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE

# ===== 第3步：把这些新规则更新到 firewall.user，保证重启不丢 =====
cat > /etc/firewall.user << 'EOF'
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0.2 -d 10.0.0.0/8     -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -d 172.16.0.0/12  -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -d 192.168.0.0/16 -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
EOF

chmod +x /etc/firewall.user
/etc/init.d/firewall restart

```

# **验证：**

```bash
# 1. NAT 规则顺序（内网 ACCEPT 在前，MASQUERADE 在最后）
iptables -t nat -L POSTROUTING -n --line-numbers

# 2. TTL 规则是否在
iptables -t mangle -L POSTROUTING -n | grep TTL

# 3. IP 转发是否开启
cat /proc/sys/net/ipv4/ip_forward

```

```
预期结果：
- NAT 表：先看到 3 条 ACCEPT（10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16），最后一条 MASQUERADE
- mangle 表：有一条 TTL set to 128
- ip_forward 显示 1

三条都符合就说明一切正常。

```



## 补充一点，本身路由器固件是自带NTP的，默认设置如图
![NTP设置](screenshot3.png)
> [!CAUTION]
>### 基本的上网是解决了，现在有一个问题就是无法访问我们学校的内网系统，目前还不知道怎么解决，解决了我会更新README的

# 报错日志1：
## 背景：主播把路由器从宿舍搬到的实验室（主要是路由器后台WAN口设置的静态ip所有参数没有变），然后到了实验室路由器就不可用了
## 问题根源：路由器 WAN 口「静态 IP」，且未正确匹配学校网络的 MAC/IP 绑定策略，导致学校拦截路由器流量，出现 “什么都 ping 不通、访问不了深澜 172.19.0.5”，且改回 DHCP 后，NAT（共享上网）/TTL（防检测）核心规则仅写入防火墙文件但未手动执行，规则未生效，依然无法连通。
## 解决办法： 
## 1.将 WAN 口从 “静态 IP” 改回 “DHCP 客户端”，恢复路由器自身 MAC，让路由器重新从学校 DHCP 服务器获取合法 IP；
## 2. 手动执行 NAT/TTL 规则（临时生效）：
```bash
iptables -t nat -A POSTROUTING -o eth0.2 -j MASQUERADE
iptables -t mangle -A POSTROUTING -o eth0.2 -j TTL --ttl-set 128
echo 1 > /proc/sys/net/ipv4/ip_forward
```
## 3.重启防火墙（确保规则永久生效）：
```bash
/etc/init.d/firewall restart
```
## 验证：

```bash
#  验证核心规则是否生效（均显示1条）
echo "NAT规则数：$(iptables -t nat -L POSTROUTING -n | grep MASQUERADE | wc -l)"
echo "TTL规则数：$(iptables -t mangle -L POSTROUTING -n | grep TTL | wc -l)"
echo "IP转发状态：$(cat /proc/sys/net/ipv4/ip_forward)" # 显示1
```
```
#  验证网络连通性（能ping通即正常）
ping -c 4 172.19.0.5  # 深澜认证页
ping -c 4 www.baidu.com  # 外网
```


> [!TIP]
> ## <span style="color: #22c55e;">! 又报错了</span>
> <span style="color: #22c55e;">我知道，那你说怎么办呢</span>

> [!CAUTION]
> ## <span style="color: #ef4444;">我乱改一通，居然过了，嘿嘿嘿</span>

> [!CAUTION]
> ## <span style="color: #ef4444;">又要没完没了的STFW了？</span>
> <span style="color: #ef4444;">对</span>

## 送大家一句话，不要问我经历了什么：
> [!CAUTION]
> *不要随便执行AI助手给的删除命令*
> 
## 整理By：Mika




echo "10.1.20.97 auth.sztu.edu.cn" >> /etc/hosts
/etc/init.d/dnsmasq restart