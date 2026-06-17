# 搬瓦工穿透完整教程：用搬瓦工 VPS 跑 frp 内网穿透，从选机器到配置上线一篇搞定——适合没公网 IP 的家用 NAS、远程桌面、SSH 等多种场景

---

家里的 NAS 突然想在公司访问，打开手机发现根本没有公网 IP。试过几个免费穿透工具，限速、断线、随时关服务，折腾了一个晚上没跑起来。后来转用搬瓦工 VPS 自建 frp，第一次连上来的那一刻确实有点小感动——延迟低，带宽稳，自己掌控。这篇把整个过程写下来，包括怎么选套餐、怎么配置，踩过的坑也顺便说说。

---

## 搬瓦工穿透是什么，为什么选它做服务端

**内网穿透**，简单说就是让没有公网 IP 的设备（家里的电脑、NAS、树莓派）被外网访问到。实现原理不复杂：需要一台有固定公网 IP 的服务器做中转，frp 这类工具就是负责把流量从公网服务器转发到你的内网设备上。

**搬瓦工（BandwagonHost）**是一家做了二十来年的 VPS 商家，隶属加拿大 IT7 Networks，在国内用户圈子里口碑很稳。拿它做穿透服务端有几个实际优势：

- 分配固定独立 IPv4，公网地址不会变
- 提供 CN2 GIA、CN2 GT 等中国优化线路，从国内访问延迟低
- KiwiVM 后台可以一键重装系统、重启服务，不用找客服
- 支持支付宝付款，国内用户购买不麻烦

用第三方免费穿透服务的问题在于——你用的服务器不是你的，随时可能断，带宽也是被限死的。自己搭一台搬瓦工，稳定性完全不一样。

---

## 搬瓦工哪个套餐适合做穿透服务端

穿透这个场景对配置要求其实不高。frp 本身很轻，一个 1 核 512MB 的机器跑起来毫无压力。真正影响体验的是**线路质量**，也就是从你家到 VPS 的延迟和稳定性。

以下是搬瓦工目前常规在售的几个套餐系列，按预算从低到高：

| 套餐系列 | 内存 | 月流量 | 价格 | 可选机房 | 购买链接 |
|----------|------|--------|------|----------|---------|
| KVM Basic A | 1 GB | 1 TB | $49.99/年 | 洛杉矶等欧美多机房 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=57) |
| KVM Basic B | 1 GB | 2 TB | $99.99/年 | 洛杉矶等欧美多机房 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=58) |
| CN2 GIA-E（E-Commerce）A | 1 GB | 1 TB | $49.99/季，$169.99/年 | DC6、DC9、日本软银、荷兰 9929 等 12 个机房 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=87) |
| CN2 GIA-E（E-Commerce）B | 2 GB | 2 TB | $89.99/季，$299.99/年 | 同上 12 个机房 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=88) |
| 香港 CN2 GIA（Ultra）A | 2 GB | 500 GB | $89.99/月，$899.99/年 | 香港 HKHK_8 + 多个亚太高端机房 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=94) |
| 香港 CN2 GIA（Ultra）B | 4 GB | 1 TB | $155.99/月，$1559.99/年 | 同上 |  [选择此方案](https://bwh81.net/aff.php?aff=80238&pid=95) |

**用来跑穿透，怎么选？**

个人轻量用途（SSH、远程桌面、偶尔访问 NAS），KVM Basic 套餐完全够。洛杉矶的机器延迟比国内接近，晚高峰稳定性取决于你是哪家运营商，电信用户可以在后台切 CN2 机房（DC3）改善不少。

如果你有多台设备要穿透、流量消耗大，或者对延迟比较在意，CN2 GIA-E 套餐是性价比比较高的选择，可用机房多，线路质量上一个档次。

香港套餐延迟确实低（广东方向能压到 10ms 出头），但价格不适合只用来穿透——除非你本来就有其他高需求场景。

👉 [查看搬瓦工当前所有套餐与价格](https://bit.ly/BanWaGon)

---

## 搬瓦工穿透环境准备

买好 VPS 后，先做几件事：

1. 登录 KiwiVM 后台，确认系统版本（建议选 Debian 11 或 Ubuntu 22.04）
2. 记下 VPS 的公网 IP 地址
3. 用 SSH 工具（Windows 可以用 PuTTY 或者直接 Terminal）连上去
4. 开放防火墙端口——稍后配置 frp 需要用到几个端口

端口这件事很多人忘了，配完 frp 一直连不上，最后发现是防火墙没放行。搬瓦工的机器 Ubuntu 系统默认 ufw 是不开启状态，Debian 也类似，但有些情况会有 iptables 规则，建议配置 frp 之前先确认一下：

bash
# 查看 iptables 规则
iptables -L -n


没什么限制的话就正常继续。如果发现有规则挡着，根据实际情况处理。

---

## frp 安装配置教程（搬瓦工服务端 + 本地客户端）

frp 分两个部分：运行在搬瓦工 VPS 上的 **frps（服务端）**，和运行在你本地内网设备上的 **frpc（客户端）**。

### 第一步：在搬瓦工 VPS 上部署 frps

SSH 连上 VPS 后，先查一下系统架构：

bash
arch


输出 `x86_64` 的话，对应下载 `amd64` 版本。去 frp 的 GitHub Releases 页面下载对应版本，比如：

bash
wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
tar -zxvf frp_0.62.0_linux_amd64.tar.gz
cd frp_0.62.0_linux_amd64


解压后你会看到 `frps` 和 `frpc` 两个可执行文件，以及对应的配置文件。服务端只需要 `frps`。

创建 frps 的配置文件（现在版本用 toml 格式，不再是旧的 ini）：

bash
nano frps.toml


最简配置如下：

toml
bindPort = 7000
auth.token = "你自己设置一个复杂的密码"


`bindPort` 是 frps 监听的端口，客户端连接服务端用这个。`auth.token` 是连接鉴权，客户端配置里要保持一致。

保存后启动：

bash
./frps -c frps.toml


看到类似 `start frps success` 的提示就说明跑起来了。

### 第二步：设置 frps 开机自启（systemd）

生产环境还是要做一下持久化，不然 VPS 重启之后要手动再拉起来：

bash
# 把 frps 复制到系统路径
cp frps /usr/local/bin/

# 创建 systemd 服务文件
nano /etc/systemd/system/frps.service


填入：

ini
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target


然后把配置文件放过去：

bash
mkdir /etc/frp
cp frps.toml /etc/frp/

systemctl daemon-reload
systemctl enable frps
systemctl start frps
systemctl status frps


status 显示 active (running) 就对了。

### 第三步：开放 VPS 防火墙端口

frps 监听的端口（7000）和后面穿透要用的端口都需要放行。以 ufw 为例：

bash
ufw allow 7000/tcp
# 如果要穿透 SSH（22端口），还需要放行穿透目标端口，比如 6000
ufw allow 6000/tcp


如果没用 ufw，用 iptables 处理：

bash
iptables -I INPUT -p tcp --dport 7000 -j ACCEPT
iptables -I INPUT -p tcp --dport 6000 -j ACCEPT


### 第四步：在本地设备上部署 frpc

把 frp 包下载到你要穿透的内网设备上（Windows、Linux、Mac 都有对应版本），编辑 `frpc.toml`：

toml
serverAddr = "你的搬瓦工 VPS 公网 IP"
serverPort = 7000
auth.token = "和服务端一样的密码"

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000


这个配置的意思是：把本地的 22 端口（SSH）映射到 VPS 的 6000 端口。外网访问 `VPS_IP:6000` 就等于 SSH 到你的内网设备。

启动客户端：

bash
./frpc -c frpc.toml


看到 `login to server success` 就通了。

---

## 实测连接效果

配置完之后从另一台外网机器测试，SSH 连接 `VPS_IP:6000`。延迟比直连快了很多，原因是搬瓦工 CN2 线路到国内本身就有优化。用下来感觉稳，跑了两三周没遇到自动断线的情况。

文件传输这块，走 frp TCP 穿透的速度基本取决于搬瓦工套餐的带宽上限。KVM Basic 套餐是 1Gbps 的带宽，实际跑起来受国内出口带宽限制，不会真的跑满，但日常文件拉取和远程操作没问题。

如果你要跑远程桌面（RDP），配置方法一样，把 `localPort` 改成 `3389`，再分配一个 remotePort 就行。HTTP 服务的话用 `type = "http"` 模式更合适，frp 对 HTTP/HTTPS 穿透也有原生支持。

不夸张，自建完之后基本忘了"公网 IP"这件事存在。

---

## 搬瓦工穿透常见问题

**Q：搬瓦工 VPS 默认有公网 IP 吗？**  
每台搬瓦工 VPS 都分配一个独立 IPv4，这正是做穿透服务端的前提。不需要额外付费，买套餐就带。

**Q：frp 连不上怎么排查？**  
先确认 frps 是否正常运行（`systemctl status frps`），再检查防火墙端口是否放行，最后对比客户端配置里的 IP、端口、token 是否和服务端完全一致。大多数连接失败都是 token 填错或者端口没开。

**Q：穿透后延迟高，有办法优化吗？**  
换成延迟更低的机房是最直接的方法。搬瓦工 KiwiVM 后台支持一键迁移机房，比如电信用户换到 DC6 CN2 GIA-E，延迟可以降不少。CN2 GIA-E 套餐支持 12 个机房自由切换，不用重新买。

**Q：frp 配置文件格式报错？**  
frp v0.50 以后已经全面迁移到 toml 格式，旧版的 `.ini` 配置不再支持。如果你搜到的教程还在用 `[common]` 那种写法，是旧版格式，照搬会报错。

**Q：搬瓦工 VPS 适合长期跑穿透吗？**  
适合。搬瓦工提供 99.9% SLA 保障，服务器稳定性没什么可挑的。我自己一台跑着好几个月了，主要就是穿透 + 轻量建站，没出过问题。

---

## 选机器直接走这里

如果你确定要买搬瓦工做穿透服务端，按预算参考：

- **轻量个人使用，预算有限**：KVM Basic $49.99/年，够用不废钱
- **多设备穿透 + 需要切机房优化延迟**：CN2 GIA-E $49.99/季起，灵活性更好
- **延迟要求高、在广东或香港附近**：Ultra 香港套餐，延迟碾压级别，但价格不是给穿透专门准备的

👉 [立即查看搬瓦工全套餐与当前优惠](https://bit.ly/BanWaGon)

购买支持支付宝直接付款，注册不复杂，买完 KiwiVM 后台就能看到 SSH 连接信息，按这篇教程走下来大概半小时内能跑通。

---

*文章仅供个人学习和合法使用参考，请确保在合法范围内使用 VPS 和穿透工具。*
