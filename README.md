# 路由器Clash翻墙实战教程

> 如何使用路由器部署Clash实现全屋设备科学上网

## 📋 目录

- [硬件准备](#硬件准备)
- [固件选择与刷机](#固件选择与刷机)
- [Clash核心配置](#clash核心配置)
- [路由器部署](#路由器部署)
- [高级功能配置](#高级功能配置)
- [监控与维护](#监控与维护)
- [故障排除](#故障排除)
- [安全与隐私](#安全与隐私)

## 🔧 硬件准备

### 推荐路由器型号

| 品牌 | 型号 | CPU | 内存 | 存储 | 价格区间 | 推荐指数 |
|------|------|-----|------|------|----------|----------|
| 小米 | AX6000 | IPQ5018 | 512MB | 128MB | 400-500元 | ⭐⭐⭐⭐⭐ |
| 斐讯 | K2P | MT7621A | 128MB | 16MB | 100-150元 | ⭐⭐⭐⭐ |
| 华硕 | RT-AX86U | BCM4908 | 1GB | 256MB | 800-1000元 | ⭐⭐⭐⭐⭐ |
| 网件 | R7800 | IPQ8065 | 512MB | 128MB | 300-400元 | ⭐⭐⭐⭐ |

### 最低配置要求

- **CPU**: MT7621A 或更高性能
- **内存**: 128MB RAM 或更多
- **存储**: 16MB Flash 或更多
- **网络**: 千兆以太网接口

## 🔄 固件选择与刷机

### 1. 固件推荐

#### OpenWrt官方固件
- **优点**: 稳定、安全、更新及时
- **缺点**: 配置复杂、软件包较少
- **适用**: 技术用户

#### ImmortalWrt
- **优点**: 基于OpenWrt，预装常用软件
- **缺点**: 更新频率较低
- **适用**: 新手用户

#### Lean's OpenWrt
- **优点**: 集成丰富插件，配置简单
- **缺点**: 非官方维护
- **适用**: 追求功能丰富的用户

### 2. 刷机步骤

#### 准备工作
```bash
# 下载固件
wget https://downloads.openwrt.org/releases/23.05.0/targets/ramips/mt7621/openwrt-23.05.0-ramips-mt7621-xiaomi_mi-router-4a-gigabit-squashfs-sysupgrade.bin

# 备份原固件
dd if=/dev/mtd0 of=/tmp/original_firmware.bin bs=1M
```

#### 刷机方法

**方法一：通过Web界面**
1. 进入路由器管理界面 (192.168.1.1)
2. 找到"系统工具" → "固件升级"
3. 选择下载的固件文件
4. 点击"开始升级"
5. 等待重启完成

**方法二：通过命令行**
```bash
# 上传固件到路由器
scp openwrt.bin root@192.168.1.1:/tmp/

# SSH登录路由器
ssh root@192.168.1.1

# 刷入固件
mtd -r write /tmp/openwrt.bin firmware
```

## ⚙️ Clash核心配置

### 1. Clash介绍

Clash是一个基于Go语言开发的多平台代理客户端，支持多种代理协议：

- **Shadowsocks (SS)**
- **ShadowsocksR (SSR)**
- **VMess (V2Ray)**
- **Trojan**
- **Hysteria**
- **WireGuard**

### 2. 核心文件结构

```
/etc/clash/
├── config.yaml          # 主配置文件
├── Country.mmdb         # GeoIP数据库
├── clash-meta           # 可执行文件
└── rules/               # 规则文件目录
    ├── gfwlist.yaml
    ├── chinalist.yaml
    └── custom.yaml
```

### 3. 基础配置文件

#### config.yaml
```yaml
# Clash配置文件
port: 7890
socks-port: 7891
allow-lan: true
mode: rule
log-level: info
external-controller: 0.0.0.0:9090

# DNS配置
dns:
  enable: true
  listen: 0.0.0.0:53
  enhanced-mode: fake-ip
  nameserver:
    - 223.5.5.5
    - 8.8.8.8
  fallback:
    - 1.1.1.1
    - 8.8.4.4

# 代理组
proxy-groups:
  - name: "自动选择"
    type: url-test
    proxies:
      - 香港节点
      - 新加坡节点
      - 美国节点
    url: 'http://www.gstatic.com/generate_204'
    interval: 300

  - name: "手动选择"
    type: select
    proxies:
      - 自动选择
      - 香港节点
      - 新加坡节点
      - 美国节点

  - name: "国外网站"
    type: select
    proxies:
      - 自动选择
      - 手动选择
      - DIRECT

# 代理节点
proxies:
  - name: "香港节点"
    type: ss
    server: hk.example.com
    port: 443
    cipher: aes-256-gcm
    password: "your_password"

  - name: "新加坡节点"
    type: vmess
    server: sg.example.com
    port: 443
    uuid: "your_uuid"
    alterId: 0
    cipher: auto
    network: ws
    ws-opts:
      path: "/path"

# 规则
rules:
  - DOMAIN-SUFFIX,google.com,国外网站
  - DOMAIN-SUFFIX,youtube.com,国外网站
  - DOMAIN-SUFFIX,netflix.com,国外网站
  - DOMAIN-SUFFIX,github.com,国外网站
  - GEOIP,CN,DIRECT
  - MATCH,国外网站
```

### 4. 订阅配置

#### 机场订阅配置
```yaml
# 在config.yaml中添加
proxy-providers:
  airport:
    type: http
    url: "https://your-airport.com/subscribe?token=your_token"
    interval: 3600
    path: ./proxies/airport.yaml
    health-check:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204

proxy-groups:
  - name: "机场节点"
    type: select
    use:
      - airport
```

#### 自建节点配置
```yaml
# 添加自建节点到proxies部分
proxies:
  - name: "自建V2Ray"
    type: vmess
    server: your-server.com
    port: 443
    uuid: "550e8400-e29b-41d4-a716-446655440000"
    alterId: 0
    cipher: auto
    network: ws
    tls: true
    ws-opts:
      path: "/v2ray"
      headers:
        Host: your-server.com
```

## 🚀 路由器部署

### 1. 安装Clash

#### 通过软件包管理器安装
```bash
# 更新软件包列表
opkg update

# 安装Clash核心
opkg install luci-app-clash

# 安装Clash Meta（推荐）
wget https://github.com/MetaCubeX/Clash.Meta/releases/download/v1.18.0/clash.meta-linux-mipsle-softfloat-v1.18.0.gz
gunzip clash.meta-linux-mipsle-softfloat-v1.18.0.gz
mv clash.meta-linux-mipsle-softfloat-v1.18.0 /usr/bin/clash-meta
chmod +x /usr/bin/clash-meta
```

#### 手动安装
```bash
# 创建目录
mkdir -p /etc/clash

# 下载Clash Meta
cd /tmp
wget https://github.com/MetaCubeX/Clash.Meta/releases/download/v1.18.0/clash.meta-linux-mipsle-softfloat-v1.18.0.gz
gunzip clash.meta-linux-mipsle-softfloat-v1.18.0.gz
mv clash.meta-linux-mipsle-softfloat-v1.18.0 /usr/bin/clash-meta
chmod +x /usr/bin/clash-meta

# 下载GeoIP数据库
wget https://github.com/Dreamacro/maxmind-geoip/releases/latest/download/Country.mmdb
mv Country.mmdb /etc/clash/
```

### 2. 配置启动脚本

#### 创建systemd服务
```bash
cat > /etc/systemd/system/clash.service << 'EOF'
[Unit]
Description=Clash daemon
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/clash-meta -d /etc/clash
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
systemctl enable clash
systemctl start clash
```

#### 创建init.d脚本
```bash
cat > /etc/init.d/clash << 'EOF'
#!/bin/sh /etc/rc.common

START=99
STOP=10

start() {
    /usr/bin/clash-meta -d /etc/clash &
}

stop() {
    killall clash-meta
}

restart() {
    stop
    start
}
EOF

chmod +x /etc/init.d/clash
/etc/init.d/clash enable
```

### 3. 网络配置

#### 配置透明代理
```bash
# 添加iptables规则
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 7890
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 7890

# 保存规则
iptables-save > /etc/iptables.rules
```

#### 配置DNS
```bash
# 修改/etc/config/dhcp
config dnsmasq
    option domainneeded '1'
    option boguspriv '1'
    option filterwin2k '0'
    option localise_queries '1'
    option rebind_protection '1'
    option rebind_localhost '1'
    option local '/lan/'
    option domain 'lan'
    option expandhosts '1'
    option nonegcache '0'
    option authoritative '1'
    option readethers '1'
    option leasefile '/tmp/dhcp.leases'
    option resolvfile '/tmp/resolv.conf.auto'
    option nonwildcard '1'
    option localservice '1'
    option cachesize '1000'
    option noresolv '1'
    option server '127.0.0.1#5353'
    option server '223.5.5.5'
    option server '8.8.8.8'
```

## 🎯 高级功能配置

### 1. 智能分流

#### 游戏加速规则
```yaml
rules:
  # Steam游戏平台
  - DOMAIN-SUFFIX,steamcommunity.com,游戏加速
  - DOMAIN-SUFFIX,steampowered.com,游戏加速
  - DOMAIN-SUFFIX,steamstatic.com,游戏加速
  
  # Epic Games
  - DOMAIN-SUFFIX,epicgames.com,游戏加速
  - DOMAIN-SUFFIX,unrealengine.com,游戏加速
  
  # 暴雪游戏
  - DOMAIN-SUFFIX,battle.net,游戏加速
  - DOMAIN-SUFFIX,blizzard.com,游戏加速

proxy-groups:
  - name: "游戏加速"
    type: url-test
    proxies:
      - 香港游戏节点
      - 新加坡游戏节点
    url: 'http://www.gstatic.com/generate_204'
    interval: 60
```

#### 流媒体解锁规则
```yaml
rules:
  # Netflix
  - DOMAIN-SUFFIX,netflix.com,流媒体解锁
  - DOMAIN-SUFFIX,nflximg.net,流媒体解锁
  - DOMAIN-SUFFIX,nflxext.com,流媒体解锁
  
  # YouTube
  - DOMAIN-SUFFIX,youtube.com,流媒体解锁
  - DOMAIN-SUFFIX,ytimg.com,流媒体解锁
  - DOMAIN-SUFFIX,googlevideo.com,流媒体解锁
  
  # Disney+
  - DOMAIN-SUFFIX,disneyplus.com,流媒体解锁
  - DOMAIN-SUFFIX,disney.com,流媒体解锁

proxy-groups:
  - name: "流媒体解锁"
    type: select
    proxies:
      - 美国流媒体节点
      - 香港流媒体节点
      - 新加坡流媒体节点
```

### 2. 广告屏蔽

#### 添加广告屏蔽规则
```yaml
rules:
  # 广告屏蔽
  - DOMAIN-SUFFIX,googlesyndication.com,REJECT
  - DOMAIN-SUFFIX,googletagmanager.com,REJECT
  - DOMAIN-SUFFIX,doubleclick.net,REJECT
  - DOMAIN-SUFFIX,facebook.com,REJECT
  - DOMAIN-SUFFIX,instagram.com,REJECT
  - DOMAIN-SUFFIX,twitter.com,REJECT
  - DOMAIN-SUFFIX,t.co,REJECT
```

### 3. 性能优化

#### 内存优化
```yaml
# 在config.yaml中添加
tun:
  enable: true
  stack: system
  dns-hijack:
    - 8.8.8.8:53
    - 1.1.1.1:53
  auto-route: true
  auto-detect-interface: true

# 减少内存使用
experimental:
  ignore-resolve-fail: true
  sniff-tls-sni: true
```

#### 并发优化
```yaml
# 调整并发连接数
tun:
  enable: true
  stack: system
  dns-hijack:
    - 8.8.8.8:53
  auto-route: true
  auto-detect-interface: true
  strict-route: true
```

## 📊 监控与维护

### 1. 状态监控

#### 实时状态查看
```bash
# 查看Clash状态
systemctl status clash

# 查看端口占用
netstat -tlnp | grep clash

# 查看进程
ps aux | grep clash

# 查看日志
journalctl -u clash -f
```

#### 流量统计
```bash
# 查看网络接口流量
cat /proc/net/dev

# 查看Clash API统计
curl http://127.0.0.1:9090/stats
```

### 2. 配置管理

#### 配置备份
```bash
#!/bin/bash
# 备份脚本
BACKUP_DIR="/tmp/clash_backup"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR
cp -r /etc/clash $BACKUP_DIR/clash_$DATE
tar -czf /tmp/clash_backup_$DATE.tar.gz -C $BACKUP_DIR clash_$DATE
echo "备份完成: /tmp/clash_backup_$DATE.tar.gz"
```

#### 配置恢复
```bash
#!/bin/bash
# 恢复脚本
BACKUP_FILE=$1
if [ -z "$BACKUP_FILE" ]; then
    echo "使用方法: $0 <备份文件路径>"
    exit 1
fi

tar -xzf $BACKUP_FILE -C /tmp/
BACKUP_DIR=$(tar -tzf $BACKUP_FILE | head -1 | cut -f1 -d"/")
cp -r /tmp/$BACKUP_DIR/* /etc/clash/
systemctl restart clash
echo "配置恢复完成"
```

### 3. 自动更新

#### 订阅自动更新
```bash
#!/bin/bash
# 订阅更新脚本
cd /etc/clash

# 下载最新配置
wget -O config_new.yaml "https://your-airport.com/subscribe?token=your_token"

# 验证配置文件
/usr/bin/clash-meta -t -d /etc/clash -f config_new.yaml

if [ $? -eq 0 ]; then
    mv config_new.yaml config.yaml
    systemctl reload clash
    echo "$(date): 配置更新成功" >> /var/log/clash_update.log
else
    echo "$(date): 配置更新失败" >> /var/log/clash_update.log
    rm config_new.yaml
fi
```

#### 设置定时任务
```bash
# 编辑crontab
crontab -e

# 添加定时任务（每小时更新一次）
0 * * * * /etc/clash/update_subscription.sh
```

## 🔧 故障排除

### 1. 常见问题

#### 问题1：Clash无法启动
```bash
# 检查配置文件语法
clash-meta -t -d /etc/clash

# 检查端口占用
netstat -tlnp | grep 7890

# 查看详细日志
journalctl -u clash --no-pager
```

**解决方案：**
1. 检查配置文件YAML语法
2. 确保端口未被占用
3. 检查文件权限

#### 问题2：无法访问外网
```bash
# 检查DNS解析
nslookup google.com 127.0.0.1

# 检查代理连接
curl -x http://127.0.0.1:7890 http://www.google.com

# 检查iptables规则
iptables -t nat -L
```

**解决方案：**
1. 检查DNS配置
2. 验证代理节点可用性
3. 检查防火墙规则

#### 问题3：速度慢
```bash
# 测试节点延迟
curl -o /dev/null -s -w "时间: %{time_total}s\n" http://www.google.com

# 检查CPU使用率
top -p $(pgrep clash-meta)

# 检查内存使用
free -h
```

**解决方案：**
1. 更换更快的节点
2. 调整并发连接数
3. 优化路由器性能

### 2. 调试工具

#### 网络诊断脚本
```bash
#!/bin/bash
# 网络诊断脚本
echo "=== Clash状态检查 ==="
systemctl is-active clash

echo "=== 端口检查 ==="
netstat -tlnp | grep -E "(7890|7891|9090)"

echo "=== DNS测试 ==="
nslookup google.com 127.0.0.1

echo "=== 代理测试 ==="
curl -x http://127.0.0.1:7890 -s -o /dev/null -w "HTTP代理: %{http_code}\n" http://www.google.com
curl -x socks5://127.0.0.1:7891 -s -o /dev/null -w "SOCKS5代理: %{http_code}\n" http://www.google.com

echo "=== 节点测试 ==="
curl -s http://127.0.0.1:9090/proxies | jq '.proxies | keys[]'
```

#### 性能监控脚本
```bash
#!/bin/bash
# 性能监控脚本
while true; do
    echo "$(date): CPU: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)%, 内存: $(free | grep Mem | awk '{printf "%.1f%%", $3/$2 * 100.0}')"
    sleep 60
done
```

## 🔒 安全与隐私

### 1. 安全配置

#### 防火墙规则
```bash
# 只允许内网访问管理端口
iptables -A INPUT -p tcp --dport 9090 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP

# 限制SSH访问
iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

#### 访问控制
```yaml
# 在config.yaml中配置
external-controller: 127.0.0.1:9090
secret: "your_secret_key"
```

### 2. 隐私保护

#### DNS泄露防护
```yaml
dns:
  enable: true
  listen: 0.0.0.0:53
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - 223.5.5.5
    - 8.8.8.8
  fallback:
    - 1.1.1.1
    - 8.8.4.4
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4
      - 0.0.0.0/32
```

#### WebRTC泄露防护
```bash
# 添加iptables规则阻止WebRTC
iptables -A FORWARD -p udp --dport 3478 -j DROP
iptables -A FORWARD -p udp --dport 5349 -j DROP
iptables -A FORWARD -p tcp --dport 3478 -j DROP
iptables -A FORWARD -p tcp --dport 5349 -j DROP
```

### 3. 日志安全

#### 日志轮转配置
```bash
# 创建logrotate配置
cat > /etc/logrotate.d/clash << 'EOF'
/var/log/clash.log {
    daily
    missingok
    rotate 7
    compress
    notifempty
    create 644 root root
    postrotate
        systemctl reload clash
    endscript
}
EOF
```

#### 敏感信息过滤
```bash
# 创建日志过滤脚本
cat > /usr/local/bin/clash_log_filter.sh << 'EOF'
#!/bin/bash
# 过滤敏感信息
sed -i 's/password=[^[:space:]]*/password=***/g' /var/log/clash.log
sed -i 's/token=[^[:space:]]*/token=***/g' /var/log/clash.log
sed -i 's/uuid=[^[:space:]]*/uuid=***/g' /var/log/clash.log
EOF

chmod +x /usr/local/bin/clash_log_filter.sh

# 添加到定时任务
echo "0 2 * * * /usr/local/bin/clash_log_filter.sh" | crontab -
```

## 📚 附录

### 1. 常用命令

```bash
# 启动Clash
systemctl start clash

# 停止Clash
systemctl stop clash

# 重启Clash
systemctl restart clash

# 查看状态
systemctl status clash

# 查看日志
journalctl -u clash -f

# 重载配置
systemctl reload clash

# 测试配置
clash-meta -t -d /etc/clash
```

### 2. 配置文件模板

#### 最小配置模板
```yaml
port: 7890
socks-port: 7891
allow-lan: true
mode: rule
log-level: info
external-controller: 0.0.0.0:9090

dns:
  enable: true
  listen: 0.0.0.0:53
  enhanced-mode: fake-ip
  nameserver:
    - 223.5.5.5
    - 8.8.8.8

proxy-groups:
  - name: "自动选择"
    type: url-test
    proxies:
      - 节点1
      - 节点2
    url: 'http://www.gstatic.com/generate_204'
    interval: 300

proxies:
  - name: "节点1"
    type: ss
    server: server1.com
    port: 443
    cipher: aes-256-gcm
    password: "password1"

rules:
  - DOMAIN-SUFFIX,google.com,自动选择
  - GEOIP,CN,DIRECT
  - MATCH,自动选择
```

### 3. 故障排除检查清单

- [ ] 路由器固件是否正确安装
- [ ] Clash核心文件是否存在且可执行
- [ ] 配置文件语法是否正确
- [ ] 代理节点是否可用
- [ ] 端口是否被占用
- [ ] 防火墙规则是否正确
- [ ] DNS配置是否正确
- [ ] 网络接口是否正常
- [ ] 日志中是否有错误信息
- [ ] 系统资源是否充足

### 4. 性能优化建议

1. **硬件优化**
   - 选择性能更强的路由器
   - 增加内存和存储空间
   - 使用千兆网络接口

2. **软件优化**
   - 使用Clash Meta版本
   - 启用TUN模式
   - 优化DNS配置
   - 调整并发连接数

3. **网络优化**
   - 选择延迟低的节点
   - 使用CDN加速
   - 优化分流规则
   - 启用流量压缩

---

## ⚠️ 免责声明

本教程仅供学习和研究使用，请遵守当地法律法规。使用本教程所产生的任何后果，作者不承担任何责任。请合理使用网络资源，尊重他人的知识产权。

## 📞 技术支持

如果您在使用过程中遇到问题，可以通过以下方式获取帮助：

1. 查看本文档的故障排除部分
2. 搜索相关技术论坛
3. 查看Clash官方文档
4. 提交Issue到项目仓库

---

**最后更新**: 2024年1月  
**版本**: v1.0  
**作者**: Router VPN Team
