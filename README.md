# Linux TCP+UDP 透明代理
## 脚本简介
ss-tproxy v3 是 ss-tproxy v2 的精简优化版，v3 版本去掉了很多不是那么常用的代理模式，如 tun2socks、tcponly，并提取出了 ss/ssr/v2ray 等代理软件的相同规则，所以 v3 版本目前只有两大代理模式：REDIRECT + TPROXY、TPROXY + TPROXY（纯 TPROXY 方式）。REDIRECT + TPROXY 是指 TCP 使用 REDIRECT 方式代理而 UDP 使用 TPROXY 方式代理；纯 TPROXY 方式则是指 TCP 和 UDP 均使用 TPROXY 方式代理。目前来说，ss-libev、ssr-libev、v2ray-core、redsocks2 均为 REDIRECT + TPROXY 组合方式，而最新版 v2ray-core 则支持纯 TPROXY 方式的代理。在 v3 中，究竟使用哪种组合是由 `proxy_tproxy='boolean_value'` 决定的，如果为 true 则为纯 TPROXY 模式，否则为 REDIRECT + TPROXY 模式（默认）。

v3 版本仍然实现了 global、gfwlist、chnonly、chnroute 四种分流模式；global 是指全部流量都走代理；gfwlist 是指 gfwlist.txt 与 gfwlist.ext 列表中的地址走代理，其余走直连；chnonly 本质与 gfwlist 没区别，只是 gfwlist.txt 与 gfwlist.ext 列表中的域名为大陆域名，所以 chnonly 是国外翻回国内的专用模式；chnroute 则是从 v1 版本开始就有的模式，也就是大家熟知的绕过局域网和大陆地址段模式，所以只要是发往国外地址的流量都会走代理出去，这也是 ss-tproxy v3 的默认模式。

ss-tproxy 可以运行在 Linux 软路由/网关、Linux 物理机、Linux 虚拟机等环境中，可以透明代理 ss-tproxy 主机本身以及所有网关指向 ss-tproxy 主机的其它主机的 TCP 与 UDP 流量。即使 ss-tproxy 不是运行在 Linux 软路由/网关上，但通过某些"技巧"，ss-tproxy 依旧能够透明代理其它主机的 TCP 与 UDP 流量。比如你在某台内网主机（假设 IP 地址为 192.168.0.100）中运行 ss-tproxy，那么你只要将该内网中的其它主机的网关以及 DNS 服务器设为 192.168.0.100，那么这些内网主机的 TCP 和 UDP 就会被透明代理。当然这台内网主机也可以是一个 Linux 虚拟机（网络要设为桥接模式，只需要一张网卡）。



## 脚本依赖
- [ss-tproxy 脚本相关依赖的安装方式参考](https://www.zfl9.com/ss-redir.html#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96)
- global 模式：TPROXY 模块、ip 命令、dnsmasq 命令
- gfwlist 模式：TPROXY 模块、ip 命令、dnsmasq 命令、perl 命令、ipset 命令
- chnroute 模式：TPROXY 模块、ip 命令、dnsmasq 命令、chinadns 命令、ipset 命令

## 端口占用
- global 模式：dnsmasq:60053@tcp+udp
- gfwlist 模式：dnsmasq:60053@tcp+udp
- chnroute 模式：dnsmasq:60053@tcp+udp、chinadns:65353@udp

> 注意：只要当前系统中的其它 dnsmasq 进程不监听 60053 端口，就没有任何影响。

## 脚本用法
**安装**
```bash
git clone https://github.com/zfl9/ss-tproxy
cd ss-tproxy
cp -af ss-tproxy /usr/local/bin
chmod 0755 /usr/local/bin/ss-tproxy
chown root:root /usr/local/bin/ss-tproxy
mkdir -m 0755 -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist.* chnroute.* /etc/ss-tproxy
chmod 0644 /etc/ss-tproxy/* && chown -R root:root /etc/ss-tproxy
```

**删除**
```bash
ss-tproxy stop
ss-tproxy flush-iptables
rm -fr /etc/ss-tproxy /usr/local/bin/ss-tproxy
```

**简介**
- `ss-tproxy`：脚本文件
- `ss-tproxy.conf`：配置文件
- `ss-tproxy.service`：服务文件
- `gfwlist.txt`：gfwlist 域名文件，不可配置
- `gfwlist.ext`：gfwlsit 黑名单文件，可配置
- `chnroute.set`：chnroute for ipset，不可配置
- `chnroute.txt`：chnroute for chinadns，不可配置

**配置**
- 脚本配置文件为 `/etc/ss-tproxy/ss-tproxy.conf`，修改后重启脚本才能生效
- 默认分流模式为 `chnroute`，这也是 v1 版本中的分流模式，根据你的需要修改
- 根据实际情况，修改 `proxy` 配置段中的代理软件的信息，详细内容见下面的说明
- `dns_remote` 为远程 DNS 服务器（走代理），默认为 Google DNS，根据需要修改
- `dns_direct` 为直连 DNS 服务器（走直连），默认为 114 公共DNS，根据需要修改
- `iptables_intranet` 为要代理的内网的网段，默认为 192.168.0.0/16，根据需要修改
- 如需配置 gfwlist 扩展列表，请编辑 `/etc/ss-tproxy/gfwlist.ext`，然后重启脚本生效

> 注意，`iptables_intranet` 不允许省略后面的 `0`，如 `192.168/16` 是错误的，请规范填写。

`proxy_server` 用来填写服务器的地址，可以是域名也可以是 IP，支持填写多个服务器的地址，使用空格隔开就行。这里解释一下多个服务器地址的作用，其实这个功能是最近才加上去的，也是受到了某位热心网友的启发，在这之前，proxy_server 只能填写一个地址，但是有些时候我们经常需要切换代理服务器，比如现在我手中有 A、B 两台服务器，目前用的是 A 服务器做代理，但是因为某些不可抗拒的因素，A 服务器出了点问题，我需要切换到 B 服务器来上网，那么必须修改 ss-tproxy.conf 里面的 proxy_server，然后修改对应的启动命令以及关闭命令，最后才能执行 `ss-tproxy restart` 来生效，然后过了段时间，发现 A 服务器好了，因为 A 服务器的线路比 B 服务器的好，所以我又想切换回 A 服务器，这时候又要重复上述步骤，改完配置文件再重启 ss-tproxy，非常麻烦。

为了解决这个问题，我将 `proxy_server` 改了一下，让它支持填写多个地址（空格隔开），那么支持填写多个服务器地址又有什么用呢？又是如何解决上述问题的呢？还是以上面的例子为例，我现在将 A、B 两台服务器的地址都填写到 proxy_server 中，默认先使用 A 服务器，然后执行 `ss-tproxy start` 启动代理；那么现在要切换为 B 服务器该如何做呢？很简单，你只需要停止之前的 A 服务器代理进程（假设为 ss-redir，且假设使用 systemctl 管理），即 `systemctl stop ss-redir@A`、`systemctl start ss-redir@B`，就行了，你不需要操作 ss-tproxy 的任何东西，就完成了代理服务器的切换。同理，如果有 5 个常用服务器，也都可以写到 proxy_server 里面，这样 ss-tproxy 启动后基本就不用去管它了，随意切换代理。

`proxy_dports` 用来填写要放行的服务器端口，默认为空，表示所有服务器端口都放行。如果你需要修改此配置，请记得将当前使用的服务器端口给放行（也就是 ss、ssr、v2ray 服务器的监听端口），否则会出现死循环。这个选项也是最近才添加的，原先版本中，默认也是将所有服务器端口都放行，但我最近使用 scp 向 vps 传输文件的时候总是会被 gfw 干扰（没几秒就显示 `stalled`），烦的很，所以就加了这个选项。这个选项的值会被作为 iptables multiport 模块的参数，所以格式为：`port[,port:port,port...]`（方括号和 `...` 不要输进去，这只是格式说明）。比如我的 ss 监听端口为 443，就写 `proxy_dports='443'`；又比如我的 v2ray 监听端口为 1000:2000（动态端口范围），并且我还想放行 80 和 443 端口，就写：`proxy_dports='80,443,1000:2000'`。另外注意，这个选项对 gfwlist 分流模式是没有效果的。

`proxy_runcmd` 是用来启动代理软件的命令，此命令不可以占用前台（意思是说这个命令必须能够立即返回），否则 `ss-tproxy start` 将被阻塞；`proxy_kilcmd` 是用来停止代理软件的命令。`proxy_runcmd` 和 `proxy_kilcmd` 的常见的写法有：
```bash
# runcmd
command args...
service srvname start
systemctl start srvname
/path/to/start.proxy.script
(command args... </dev/null &>>/var/log/proc.log &)
setsid command args... </dev/null &>>/var/log/proc.log
nohup command args... </dev/null &>>/var/log/proc.log &
command args... </dev/null &>>/var/log/proc.log & disown

# kilcmd
pkill -9 command
service srvname stop
systemctl stop srvname
kill -9 $(pidof command)

# example
service v2ray start
systemctl start v2ray
systemctl start ss-redir
systemctl start ssr-redir
(ss-redir args... </dev/null &>>/var/log/ss-redir.log &)
(ssr-redir args... </dev/null &>>/var/log/ssr-redir.log &)

# ss-redir args
-s <server_addr>    # 服务器地址
-p <server_port>    # 服务器端口
-m <server_method>  # 加密方式
-k <server_passwd>  # 用户密码
-b <listen_addr>    # 监听地址
-l <listen_port>    # 监听端口
--no-delay          # TCP_NODELAY
--fast-open         # TCP_FASTOPEN
--reuse-port        # SO_REUSEPORT
-u                  # 启用 udp relay
-v                  # 启用详细日志输出

# ssr-redir args
-s <server_addr>    # 服务器地址
-p <server_port>    # 服务器端口
-m <server_method>  # 加密方式
-k <server_passwd>  # 用户密码
-b <listen_addr>    # 监听地址
-l <listen_port>    # 监听端口
-O <protocol>       # 协议插件
-G <protocol_param> # 协议参数
-o <obfs>           # 混淆插件
-g <obfs_param>     # 混淆参数
-u                  # 启用 udp relay
-v                  # 启用详细日志输出
```

如果还是不清楚怎么写，我再举几个具体的例子：
```bash
# ss-libev 透明代理
# 假设服务器信息如下:
# 服务器地址: ss.net
# 服务器端口: 8080
# 加密方式:   aes-128-gcm
# 用户密码:   passwd.ss.net
# 监听地址:   0.0.0.0
# 监听端口:   60080
# proxy_runcmd 如下:
(ss-redir -s ss.net -p 8080 -m aes-128-gcm -k passwd.ss.net -b 0.0.0.0 -l 60080 -u --reuse-port --no-delay --fast-open </dev/null &>>/var/log/ss-redir.log &)
# proxy_kilcmd 如下:
kill -9 $(pidof ss-redir)

# ssr-libev 透明代理
# 假设服务器信息如下:
# 服务器地址: ss.net
# 服务器端口: 8080
# 加密方式:   aes-128-cfb
# 用户密码:   passwd.ss.net
# 协议插件:   auth_chain_a
# 协议参数:   2333:protocol_param
# 混淆插件:   http_simple
# 混淆参数:   www.bing.com
# 监听地址:   0.0.0.0
# 监听端口:   60080
# proxy_runcmd 如下:
# 如果没有协议参数、混淆参数，则去掉 -G、-g 选项
(ssr-redir -s ss.net -p 8080 -m aes-128-cfb -k passwd.ss.net -O auth_chain_a -G 2333:protocol_param -o http_simple -g www.bing.com -b 0.0.0.0 -l 60080 -u </dev/null &>>/var/log/ssr-redir.log &)
# proxy_kilcmd 如下:
kill -9 $(pidof ssr-redir)

# v2ray 透明代理:
# 对于 Systemd 发行版
proxy_runcmd='systemctl start v2ray'
proxy_kilcmd='systemctl stop  v2ray'
# 对于 SysVinit 发行版
proxy_runcmd='service v2ray start'
proxy_kilcmd='service v2ray stop'
```
对于 ss-redir/ssr-redir，也可以将配置放到 json 文件，然后使用选项 `-c /path/to/config.json` 替代那一大堆参数。<br>
特别注意，ss-redir、ssr-redir 的监听地址必须要设置为 0.0.0.0（即 `-b 0.0.0.0`），不能为 127.0.0.1，也不能省略。

如果你使用的是 v2ray（此处的配置仅适用于 `2018.11.05 v4.1` 版本之后的 v2ray，含 v4.1 版本），那么你需要像下面这样配置 v2ray 客户端的 config.json（只需关注 `inbounds` 配置段，其它配置与 ss-tproxy 的使用无关），在下面这个例子中，代理方式为 REDIRECT + TPROXY（ss-tproxy 默认代理方式），如果你需要使用纯 TPROXY 代理方式，请将 `"tproxy": "redirect"` 这行注释掉，然后取消 `"tproxy": "tproxy"` 这行的注释，并且将 ss-tproxy.conf 里面的 `proxy_tproxy` 选项改为 true。
```javascript
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },

  "inbounds": [
    {
      "protocol": "dokodemo-door",
      "listen": "0.0.0.0",
      "port": 60080,
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "streamSettings": {
        "sockopt": {
          //"tproxy": "tproxy" // tproxy + tproxy
          "tproxy": "redirect" // redirect + tproxy
        }
      }
    }
  ],

  "outbounds": [
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "node.proxy.net", // server addr
            "port": 12345,               // server port
            "method": "aes-128-gcm",     // server method
            "password": "password"       // server passwd
          }
        ]
      }
    }
  ]
}
```

有人反馈 v2ray 透明代理无法成功，请务必检查 v2ray 客户端和服务端的配置。这是我测试用的 v2ray 配置：

**config.json for client**
```javascript
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },

  "inbounds": [
    {
      "protocol": "dokodemo-door",
      "listen": "0.0.0.0",
      "port": 60080,
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "streamSettings": {
        "sockopt": {
          //"tproxy": "tproxy" // tproxy + tproxy
          "tproxy": "redirect" // redirect + tproxy
        }
      }
    }
  ],

  "outbounds": [
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "node.proxy.net", // VPS 地址
            "port": 12345,               // VPS 端口
            "method": "aes-128-gcm",     // 加密方式
            "password": "password"       // 用户密码
          }
        ]
      }
    }
  ]
}
```

**config.json for server**
```javascript
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },

  "inbounds": [
    {
      "protocol": "shadowsocks",
      "address": "0.0.0.0",      // 监听地址
      "port": 12345,             // 监听端口
      "settings": {
        "method": "aes-128-gcm", // 加密方式
        "password": "password",  // 用户密码
        "network": "tcp,udp"
      }
    }
  ],

  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

如果使用 chnonly 模式（国外翻进国内），请选择 `gfwlist` mode，chnonly 模式下，你必须修改 ss-tproxy.conf 中的 `dns_remote` 为国内的 DNS，如 `dns_remote='114.114.114.114:53'`，并将 `dns_direct` 改为本地 DNS（国外的），如 `dns_direct='8.8.8.8'`；因为 chnonly 模式与 gfwlist 模式共享 gfwlist.txt、gfwlist.ext 文件，所以在第一次使用时你必须先运行 `ss-tproxy update-chnonly` 将默认的 gfwlist.txt 内容替换为大陆域名（更新列表时，也应使用 `ss-tproxy update-chnonly`），并且注释掉 gfwlist.ext 中的 Telegram IP 段，因为这是为正常翻墙设置的。要恢复 gfwlist 模式的话，请进行相反的步骤。

`dns_modify='boolean_value'`：如果值为 false（默认），则 ss-tproxy 在修改 /etc/resolv.conf 文件时，会采用 `mount -o bind` 方式（不直接修改原文件，而是“覆盖”它，在 stop 之后会自动恢复为原文件）；如果值为 true，则直接使用 I/O 重定向来修改 /etc/resolv.conf 文件。一般情况下保持默认就行，但某些时候将其设为 true 可能会好一些（具体什么时候我也不太好讲，需要具体情况具体分析，比如你使用默认的 mount 方式出现了问题，那就换为重定向方式）。

`opts_ss_netstat='auto|netstat|ss'` 选项的意思是，在检测 tcp/udp 端口时，应该使用哪个检测命令，默认为 auto，表示自动选择（如果有 ss 就使用 ss，否则使用 netstat），设为 netstat 表示使用 netstat 命令来检测，设为 ss 表示使用 ss 命令来检测。之所以添加这个选项，是因为某些系统的 ss 命令有问题，检测不到 udp 的监听端口，导致用户误以为 udp 端口没有起来。如果你也遇到了这个问题，请将该选项改为 netstat。

`ipts_non_snat='true|false'` 选项的意思是，是否需要设置 SNAT/MASQUERADE 规则，如果为 true 则表示不设置 SNAT/MASQUERADE 规则，如果为 false 则表示要设置 SNAT/MASQUERADE 规则（默认值）。如果你使用“代理网关”或者“透明桥接”模式，请将该选项改为 true，因为不需要 SNAT/MASQUERADE 规则，只有当你在“出口路由”位置运行 ss-tproxy 时才需要配置 SNAT/MASQUERADE 规则（所谓出口路由位置就是至少有两张网卡，一张连接外网，一张连接内网）。

**端口映射**

如果 ss-tproxy 运行在“代理网关”，最好将 `ipts_non_snat` 设为 true，否则端口映射必定失败（好吧，即使将其设为 true，在某些情况下端口映射依旧会失败）。我们先来简要分析一下，为什么设为 false 会导致端口映射失败。假设拨号网关为 192.168.1.1，代理网关为 192.168.1.2，内网主机为 192.168.1.100；在拨号网关上设置端口映射规则，将外网端口 8443 映射到内网主机 192.168.1.100 的 8443 端口；在代理网关上运行 ss-tproxy（假定分流模式为 gfwlist），然后将内网主机 192.168.1.100 的网关和 DNS 设为 192.168.1.2；此时代理网关以及内网主机均可透过代理来上网。

在内网主机 192.168.1.100 上运行端口为 8443 的服务进程，然后我们从其它外网主机（假设 IP 为 2.2.2.2）连接此端口上的服务。首先，外网主机向拨号网关的 8443 端口发起连接（假设 IP 为 1.1.1.1），即 `2.2.2.2:2333 -> 1.1.1.1:8443`，然后拨号网关查询到对应的端口映射规则，于是做 DNAT 转换，变为 `2.2.2.2:2333 -> 192.168.1.100:8443`，然后通过内网网卡送到了 192.168.1.100 主机的 8443 端口（SYN 握手请求成功到达）；然后服务进程会发送 SYN+ACK 握手响应包，即 `192.168.1.100:8443 -> 2.2.2.2:2333`，因为内网主机的网关为 192.168.1.2，所以 SYN+ACK 包将被送到代理网关上，因为目的地址 2.2.2.2 并没有在 gfwlist 列表中，所以放行，经过 FORWARD 链，到达 POSTROUTING 链，问题来了，ss-tproxy 已经在 POSTROUTING 链的 nat 表上设置了 SNAT 规则（`ipts_non_snat` 为 false），所以将被转换为 `192.168.1.2:6666 -> 2.2.2.2:2333`，而当这个数据包到达拨号网关时，拨号网关检查发现这个源地址并不是 192.168.1.100:8443，所以并不会按照端口映射规则将其转换为 `1.1.1.1:8443 -> 2.2.2.2:2333`，而是将其映射为一个随机端口，如 62333，所以外网主机接收到的 SYN+ACK 包的源地址是 1.1.1.1:62333，这显然是无法成功建立 TCP 连接的。

所以，对于 gfwlist 模式，只需要将 `ipts_non_snat` 设为 true，端口映射基本上就能正常工作。而对于 chnroute 模式，即使将 `ipts_non_snat` 设为了 true，在某些情况下依旧会失败，怎么说呢？比如你在 IP 为非 chnroute list 的外网主机上连接拨号网关上的映射端口，SYN 包没问题，会成功到达内网主机，但是 SYN+ACK 包在经过代理网关时，因为这个目的 IP 并不位于 chnroute list，所以会被送到代理网关上的代理进程（比如 ss-redir），也就是说这个 SYN+ACK 包会走代理出去，这显然会握手失败。如果你要让它握手成功，就必须将对应的目的 IP 放行，或者改用 gfwlist 模式。而 global 模式就不用说了，无论目的 IP 是国内还是国外，通通走代理，所以全都会握手失败，解决方法和 chnroute 模式一样，要么放行，要么用 gfwlist 模式。

但实际上，如果内网主机需要映射到外网，那么它们通常也不需要设置什么代理（即不用将网关和 dns 指向 ss-tproxy 主机），而不更改这些主机的 gateway 和 dns 自然就不会出现上述端口映射问题，因为根本不会经过 ss-tproxy，无论去程还是回程。

**桥接模式**
![桥接模式 - 网络拓扑](https://user-images.githubusercontent.com/22726048/47959326-e07fa280-e01b-11e8-95e5-32953cdbc803.png)
上图由 [@myjsqmail](https://github.com/myjsqmail) 提供，他的想法是，在不改变原网络的情况下，让 ss-tproxy 透明代理内网中的所有 TCP、UDP 流量。为了达到这个目的，他在“拨号路由”下面接了一个“桥接主机”，桥接主机有两个网口，一个连接出口路由（假设为 wan），一个连接内网总线（假设为 lan），然后将这两张网卡进行桥接，得到一个逻辑网卡（假设为 br0），在桥接主机上开启“软路由功能”，即执行 `sysctl -w net.ipv4.ip_forward=1`，然后通过 DHCP 方式，获取出口路由上分配的 IP 信息，此时，桥接主机和其它内网主机已经能够正常上网了。

然后，在桥接主机上运行 ss-tproxy，此时，桥接主机自己能够被正常代理，但是其它内网主机仍然走的直连，没有走透明代理。为什么呢？因为默认情况下，经过网桥的流量不会被 iptables 处理。所以我们必须让网桥上的流量经过 iptables 处理，首先，执行命令 `modprobe br_netfilter` 以加载 br_netfilter 内核模块，然后修改 /etc/sysctl.conf，添加：
```bash
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

net.ipv4.conf.all.route_localnet = 1
net.ipv4.conf.default.route_localnet = 1

net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
```
保存退出，然后执行 `sysctl -p` 来让这些内核参数生效。

但这还不够，我们还需要设置 ebtables 规则，首先，安装 ebtables，如 `yum -y install ebtables`，然后执行：
```bash
ebtables -t broute -A BROUTING -p IPv4 -i lan --ip-proto tcp -j redirect --redirect-target DROP
ebtables -t broute -A BROUTING -p IPv4 -i lan --ip-proto udp --ip-dport ! 53 -j redirect --redirect-target DROP
```

如果 `proxy_tproxy` 为 false，那么你还需要修改 ss-tproxy 里面的 iptables 规则，将 REDIRECT 改为 DNAT，如：
```bash
# old
iptables -t nat -A TCPCHAIN -p tcp -j REDIRECT --to-ports $proxy_tcport
# new
iptables -t nat -A TCPCHAIN -p tcp -j DNAT --to-destination 127.0.0.1:$proxy_tcport
```

没出什么意外的话，现在桥接主机和其它内网主机的 TCP 和 UDP 流量应该都是能够被 ss-tproxy 给透明代理的。<br>
还有一点，请将 `/etc/ss-tproxy/ss-tproxy.conf` 里面的 `ipts_non_snat` 选项改为 true，因为不需要 SNAT 规则。

**钩子函数**

ss-tproxy 支持 4 个钩子函数，分别是 `pre_start`（启动前执行）、`post_start`（启动后执行）、`pre_stop`（停止前执行）、`post_stop`（停止后执行）。举个例子，在不修改 ss-tproxy 脚本的前提下，设置一些额外的 iptables 规则，假设我需要在 ss-tproxy 启动后添加某些规则，然后在 ss-tproxy 停止后删除这些规则，则修改 ss-tproxy.conf，添加以下内容：
```bash
function post_start {
    iptables -A ...
    iptables -A ...
    iptables -A ...
}

function post_stop {
    iptables -D ...
    iptables -D ...
    iptables -D ...
}
```

**自启**
- `cp -af ss-tproxy.service /etc/systemd/system`
- `systemctl daemon-reload`
- `systemctl enable ss-tproxy.service`

关于自启我要多说几句，之前的开机自启是有问题的，有一定几率会失败，不过现在已经解决了这个问题，其实说起来解决方法也很简单，就是在启动之前先 ping 114.114.114.114，直到 ping 成功了才会执行 `ss-tproxy start`。经过测试，加了这个 ping 命令后，自启就没有问题了，可以确保在网络准备好之后再启动 ss-tproxy 脚本。当然，如果你使用的是 ArchLinux，也可以利用 netctl 的 hook 脚本来启动 ss-tproxy，具体做法如下：

假设你的网卡配置文件为 `/etc/netctl/eth0`（如果有多张网卡，那就选择“外网网卡”，也就是通过哪张网卡上网就选哪张网卡），进入 `/etc/netctl/hooks` 目录，创建一个空文件（即钩子文件，本质是 shell 脚本），然后给这个文件加上可执行权限（没有执行权限的钩子不会被 netctl 执行），比如我就创建一个名为 eth0.hooks 的文件（文件名随便，无要求）：
```bash
cd /etc/netctl/hooks
touch eth0.hooks
chmod +x eth0.hooks
```
然后使用你喜欢的文本编辑器打开 eth0.hooks 文件，添加以下内容：
```bash
#!/bin/bash

if [ "$Profile" = 'eth0' ]; then
    function onStart {
        /usr/local/bin/ss-tproxy start
        return 0
    }

    function onStop {
        /usr/local/bin/ss-tproxy stop
        return 0
    }

    ExecUpPost='onStart'
    ExecDownPre='onStop'
fi
```
脚本内容本身具有很好的自我解释性，我就不详细解释了，需要注意的是 `"$Profile" = 'eth0'`，因为默认情况下任何一张网卡的启动和停止都会搜寻 `/etc/netctl/hooks` 下的可执行钩子脚本，而我们实际上只需要关心 `/etc/netctl/eth0` 网卡的启动事件和关闭事件，所以就做了一下这个判断。编辑完之后保存退出，然后 reboot 测试一下是否能够正常自启动（当然你的 `/etc/netctl/eth0` 要配置自启动，即 `netctl enable eth0`）。

注意，如果你使用 `systemctl enable ss-tproxy.service` 方式配置了 ss-tproxy 的开机自启，那么应该避免直接使用 `ss-tproxy start|stop|restart` 这几个命令（当然除了这几个命令外，其它命令都是可以执行的，比如 `ss-tproxy status`、`ss-tproxy update-gfwlist`），为什么呢？因为 systemctl 启动一个脚本之后，systemctl 会在内部保存一个状态，即脚本已经 running，然后只有当你下次使用 systemctl 停止该脚本的时候，systemctl 内部才会将这个状态改为 stopped。所以配置 ss-tproxy 开机自启后，这个服务的状态就是 running，如果你执行 `ss-tproxy stop` 来停止脚本，那么这个服务状态是不会变的，依旧是 running，但实际上它已经 stopped 了，而当你执行 `systemctl start ss-tproxy` 来启动脚本时，systemctl 并不会在内部执行 `ss-tproxy start`，因为这个服务的状态是 running，说明已经启动了，就不会再次启动了。这样一来就完全混乱了，你以为执行完毕后 ss-tproxy 就启动了，然而实际上，执行 `ss-tproxy status` 看下还是 stopped 的。所以我说如果配置了 service 方式的开机自启，就不要使用 `ss-tproxy start|stop|restart` 这 3 个命令了！应使用 `systemctl start|stop|restart ss-tproxy`。

**用法**
- `ss-tproxy help`：查看帮助
- `ss-tproxy start`：启动代理
- `ss-tproxy stop`：关闭代理
- `ss-tproxy restart`：重启代理
- `ss-tproxy status`：代理状态
- `ss-tproxy check-command`：检查命令是否存在
- `ss-tproxy flush-dnscache`：清空 DNS 查询缓存
- `ss-tproxy flush-gfwlist`：清空 ipset-gfwlist IP 列表
- `ss-tproxy update-gfwlist`：更新 gfwlist（restart 生效）
- `ss-tproxy update-chnonly`：更新 chnonly（restart 生效）
- `ss-tproxy update-chnroute`：更新 chnroute（restart 生效）
- `ss-tproxy show-iptables`：查看 iptables 的 mangle、nat 表
- `ss-tproxy flush-iptables`：清空 raw、mangle、nat、filter 表

`ss-tproxy flush-gfwlist` 的作用：因为 `gfwlist` 模式下 `ss-tproxy restart`、`ss-tproxy stop; ss-tproxy start` 并不会清空 `ipset-gfwlist` 列表，所以如果你进行了 `ss-tproxy update-gfwlist`、`ss-tproxy update-chnonly` 操作，或者修改了 `/etc/tproxy/gfwlist.ext` 文件，建议在 start 前执行一下此步骤，防止因为之前遗留的 ipset-gfwlist 列表导致各种奇怪的问题。注意，如果执行了 `ss-tproxy flush-gfwlist` 那么你可能还需要清空内网主机的 dns 缓存，并重启浏览器等被代理的应用。

如果需要修改 `proxy_kilcmd`（比如将 ss 改为 ssr），请先执行 `ss-tproxy stop` 后再修改 `/etc/ss-tproxy/ss-tproxy.conf` 配置文件，否则之前的代理进程不会被 kill（因为 ss-tproxy 不可能再知道之前的 kill 命令是什么，毕竟 ss-tproxy 只是一个 shell 脚本，无法维持状态），这可能会造成端口冲突。当然也有一种取巧的办法，那就是在 proxy_kilcmd 中 kill 所有可能使用到的代理进程，比如你经常需要从 ss 切换为 ssr（或者从 ssr 切换为 ss），那么可以将 proxy_kilcmd 写为 `kill -9 $(pidof ss-redir) $(pidof ssr-redir)`，这样你就不需要先 stop 再改配置再 start 了，而是直接改好配置然后 restart。

小技巧，如果你觉得切换代理时要修改 ss-tproxy.conf 很麻烦，也可以这么做：将 proxy_runcmd 和 proxy_kilcmd 改为空调用，如 `proxy_runcmd='true'`、`proxy_kilcmd='true'`，然后配置好 proxy_server，将所有可能会用到的服务器地址都放进去，当然 proxy_dports 也可以配置好要放行的服务器端口，最后执行 `ss-tproxy start` 来启动 ss-tproxy，因为我们没有写代理进程的启动和停止命令，所以会显示代理进程未运行，没关系，现在我们要做的就是启动对应的代理进程，假设为 ss-redir 且使用 systemd 进行管理，则执行 `systemctl start ss-redir`，现在你再执行 `ss-tproxy status` 就会看到对应的状态正常了，当然代理也是正常的，如果需要换为 v2ray，假设也是使用 systemd 进行管理，那么只需要先关闭 ss-redir，然后再启动 v2ray 就行了，即 `systemctl stop ss-redir`、`systemctl start v2ray`，相当于我现在启动的只是一个代理框架，ss-tproxy 启动之后基本就不需要管它了，可以随意切换代理。

**日志**
> 脚本默认关闭了详细日志，如果需要，请修改 ss-tproxy.conf，打开相应的 log/verbose 选项

- dnsmasq：`/var/log/dnsmasq.log`
- chinadns：`/var/log/chinadns.log`

**FAQ**<br>

先说 ss/ssr 透明代理吧，ss-redir 是 ss-libev、ssr-libev 中的一个工具，配合 iptables 可以在 Linux 上实现 ss、ssr 透明代理，ss-redir 的 TCP 透明代理是通过 REDIRECT 方式实现的，而 UDP 透明代理是通过 TPROXY 方式实现的。强调一点，利用 ss-redir 实现透明代理必须使用 ss-libev 或 ssr-libev，python、go 等版本没有 ss-redir、ss-tunnel 程序。
当然，ss、ssr 透明代理并不是只能用 ss-redir 来实现，使用 ss-local + redsocks2/tun2socks 同样可以实现 socks5（ss-local 是 socks5 服务器）全局透明代理；ss-local + redsocks2 实际上是 ss-redir 的分体实现，TCP 使用 REDIRECT 方式，UDP 使用 TPROXY 方式；ss-local + tun2socks 则相当于 Android 版 SS/SSR 的 VPN 模式，因为它实际上是通过一张虚拟的 tun 网卡来进行代理的。
最后说一下 v2ray 的透明代理，其实原理和 ss/ssr-libev 一样，v2ray 可以看作是 ss-local、ss-redir、ss-tunnel、ss-server 四者的合体，因为同一个 v2ray 程序既可以作为 server 端，也可以作为 client 端。所以 v2ray 的透明代理也有两种实现方式，一是利用对应的 ss-redir + iptables，二是利用对应的 ss-local + redsocks2/tun2socks（redsocks2/tun2socks 可以与任意 socks5 代理组合，实现透明代理）。

组件区别
ss-server
shadowsocks 服务端程序，核心部件之一，各大版本均提供 ss-server 程序。

ss-local
shadowsocks 客户端程序，核心部件之一，各大版本均提供 ss-local 程序。
ss-local 是运行在本地的 socks5 代理服务器，根据 OSI 模型，socks5 是会话层协议，支持 TCP 和 UDP 的代理。

但是现在只有少数软件直接支持 socks5 代理协议，绝大多数都只支持 http 代理协议。好在我们可以利用 privoxy 将 socks5 代理转换为 http 代理，使用 privoxy 还有一个好处，那就是可以实现 gfwlist 分流模式（不过现在的 ss-tproxy 脚本也可以了），如果你对它感兴趣，可以看看 ss-local 终端代理。

ss-redir
shadowsocks-libev 提供的socks5 透明代理工具，也就是今天这篇文章的主题 - 实现透明代理！

正向代理
正向代理，即平常我们所说的代理，比如 http 代理、socks5 代理等，都属于正向代理。
正向代理的特点就是：如果需要使用正向代理访问互联网，就必须在客户端进行相应的代理设置。

透明代理
透明代理和正向代理的作用是一样的，都是为了突破某些网络限制，访问网络资源。
但是透明代理对于客户端是透明的，客户端不需要进行相应的代理设置，就能使用透明代理访问互联网。

反向代理
当然，这个不在本文的讨论范畴之内，不过既然提到了前两种代理，就顺便说说反向代理。
反向代理是针对服务端来说的，它的目的不是为了让我们突破互联网限制，而是为了实现负载均衡。

举个栗子：
ss-local 提供 socks5 正向代理，要让软件使用该代理，必须对软件进行相应的代理配置，否则不会走代理；
ss-redir 提供 socks5 透明代理，配置合适网络规则后，软件会在不知情的情况下走代理，不需要额外配置。

ss-tunnel
shadowsocks-libev 提供的本地端口转发工具，通常用于解决 dns 污染问题。

假设 ss-tunnel 监听本地端口 53，转发的远程目的地为 8.8.8.8:53；系统 dns 为 127.0.0.1。
去程：上层应用请求 dns 解析 -> ss-tunnel 接收 -> ss 隧道 -> ss-server 接收 -> 8.8.8.8:53；
回程：8.8.8.8:53 响应 dns 请求 -> ss-server 接收 -> ss 隧道 -> ss-tunnel 接收 -> 上层应用。

方案说明
用过 Linux SS/SSR 客户端（尤其指命令行界面）的都知道，它们比 Windows/Android 中的 SS/SSR 客户端难用多了，安装好就只有一个 ss-local（libev 版还有 ss-redir、ss-tunnel，但我相信大部分人装得都是 python 版的），启动 ss-local 后并不会像 Windows/Android 那样自动配置系统代理，此时它仅仅是一个本地 socks5 代理服务器，默认监听 127.0.0.1:1080，如果需要利用该 socks5 代理上外网，必须在命令中指定对应的代理，如 curl -4sSkL -x socks5h://127.0.0.1:1080 https://www.google.com。

但我想大部分人要的代理效果都不是这种的，太原始了。那能不能配置所谓的“系统代理”呢，可以是可以，但是好像只支持 http 类型的代理，即在当前 shell 中设置 http_proxy、https_proxy 环境变量，假设存在一个 http 代理（支持 CONNECT 请求方法），监听地址是 127.0.0.1:8118，可以这样做：export http_proxy=http://127.0.0.1:8118; export https_proxy=$http_proxy。执行完后，git、curl、wget 等命令会自动从环境变量中读取 http 代理信息，然后通过 http 代理连接目的服务器。

那问题来了，ss-local 提供的是 socks5 代理，不能直接使用怎么办？也简单，Linux 中有很多将 socks5 包装为 http 代理的工具，比如 privoxy。只需要在 /etc/privoxy/config 里面添加一行 forward-socks5 / 127.0.0.1:1080 .，启动 privoxy，默认监听 127.0.0.1:8118 端口，注意别搞混了，8118 是 privoxy 提供的 http 代理地址，而 1080 是 ss-local 提供的 socks5 代理地址，发往 8118 端口的数据会被 privoxy 处理并转发给 ss-local。所以我们现在可以执行 export http_proxy=http://127.0.0.1:8118; export https_proxy=$http_proxy 来配置当前终端的 http 代理，这样 git、curl、wget 这些就会自动走 ss-local 出去了。

当然我们还可以利用 privoxy 灵活的配置，实现 Windows/Android 中的 gfwlist 分流模式。gfwlist.txt 其实是对应的 Adblock Plus 规则的 base64 编码文件，显然不能直接照搬到 privoxy 上。这个问题其实已经有人解决了，利用 snachx/gfwlist2privoxy python 脚本就可轻松搞定。但其实我也重复的造了一个轮子：zfl9/gfwlist2privoxy，至于为什么要造这个轮子，是因为我当时运行不了他的脚本（也不知道什么原因），所以花了点时间用 shell 脚本实现了一个 gfwlist2privoxy（但其实我是用 perl 转换的，只不过用 shell 包装了一下）。脚本转换出来的是一个 gfwlist.action 文件，我们只需将该 gfwlist.action 文件放到 /etc/privoxy 目录，然后在 config 中添加一行 actionsfile gfwlist.action（当然之前 forward-socks5 那行要注释掉），重启 privoxy 就可以实现 gfwlist 分流了。

但仅仅依靠 http_proxy、https_proxy 环境变量实现的终端代理效果不是很好，因为有些命令根本不理会你的 http_proxy、https_proxy 变量，它们依旧走的直连。但又有大神想出了一个巧妙的方法，即 rofl0r/proxychains-ng，其原理是通过 LD_PRELOAD 特殊环境变量提前加载指定的动态库，来替换 glibc 中的同名库函数。这个 LD_PRELOAD 指向的其实就是 proxychains-ng 实现的 socket 包装库，这个包装库会读取 proxychains-ng 的配置文件（这里面配置代理信息），之后执行的所有命令调用的 socket 函数其实都是 proxychains-ng 动态库中的同名函数，于是就实现了全局代理，而命令对此一无所知。将 proxychains-ng 与 privoxy 结合起来基本上可以完美实现 ss/ssr 的本地全局 gfwlist 代理（小技巧，在 shell 中执行 exec proxychains -q bash 可以实现当前终端的全局代理，如果需要每个终端都自动全局代理，可以在 bashrc 文件中加入这行）。

但是很多人对此依然无法满足，因为他们想实现 OpenWrt 这种路由级别的全局透明代理（并且还有 gfwlist、绕过大陆地址段这些分流模式可选择），这样只要设备连到 WiFi 就能直接无缝上网，完全感觉不到“墙”的存在。如果忽略分流模式（即全部流量都走代理出去），那么实现是很简单的（几条 iptables 就可以搞定，但是这太简单粗暴了，很多国内网站走代理会非常慢，体验很不好）；但是如果要自己实现 gfwlist、绕过大陆地址段这些模式，恐怕很多人都会望而却步，因为确实复杂了一些。但这种透明代理的模式的确很诱人，毕竟只要设置一次就可以让所有内网设备上 Internet，于是我开始摸索如何在 Linux 中实现类似 OpenWrt 的代理模式，而我摸索出来的成果就是 ss-tproxy 透明代理脚本。值得说明一下，ss-tproxy 可以部署在 Linux 软路由（网关）、Linux 物理机、Linux 虚拟机等环境中；对于 Linux 软路由中的 ss-tproxy，可以用来代理网关本身以及内网主机的 TCP/UDP 流量；对于 Linux 物理机/虚拟机 中的 ss-tproxy，可以用来代理主机本身以及所有网关指向该主机的其它主机的 TCP/UDP 流量（其实就是文末的 代理网关）。

安装依赖
必要的依赖

global 模式：TPROXY 模块、ip 命令、dnsmasq 命令
gfwlist 模式：TPROXY 模块、ip 命令、dnsmasq 命令、perl 命令、ipset 命令
chnroute 模式：TPROXY 模块、ip 命令、dnsmasq 命令、chinadns 命令、ipset 命令
可选的依赖

如果要使用 ss-tproxy 的 update-gfwlist、update-chnonly、update-chnroute 功能，则还需 curl 命令
有必要声明一点，不是说下面列出的所有依赖都需要安装，你只需要装对应模式需要的依赖即可，比如你如果只用 gfwlist 模式，那么就是确保 TPROXY 模块、ip 命令、dnsmasq 命令、perl 命令、ipset 命令是否存在，如果没有那么就安装一下，仅此而已；在安装依赖前请先自己检查是否已有对应的模块或命令，不要盲目照搬照抄。很多命令其实发行版都自带了，所以实际需要的依赖项很少。

curl
请检查 curl 是否支持 HTTPS 协议，使用 curl --version 可查看（Protocols）

# CentOS
yum -y install curl

# ArchLinux
pacman -S curl
BashCopy
perl5
Perl5 的版本最好 v5.10.0+ 以上（使用 perl -v 命令可查看）

# CentOS
yum -y install perl

# ArchLinux
pacman -S perl
BashCopy
ipset
# CentOS
yum -y install ipset

# ArchLinux
pacman -S ipset
BashCopy
TPROXY
TPROXY 是一个 Linux 内核模块，在 Linux 2.6.28 后进入官方内核。一般正常的发行版都没有裁剪 TPROXY 模块，TPROXY 模块缺失问题主要出现在无线路由固件上。使用以下方法可以检测当前内核是否包含 TPROXY 模块，如果没有，请自行解决。

# 查找 TPROXY 模块
find /lib/modules/$(uname -r) -type f -name '*.ko*' | grep 'xt_TPROXY'

# 正常情况下的输出
/lib/modules/4.16.8-1-ARCH/kernel/net/netfilter/xt_TPROXY.ko.xz
BashCopy
iproute2
# CentOS
yum -y install iproute

# ArchLinux
pacman -S iproute2
BashCopy
haveged
如果有时候启动 ss-redir、ss-tunnel 会失败，且错误提示如下，则需要安装 haveged 或 rng-utils/rng-tools。虽然这个依赖是可选的，但强烈建议大家安装（并设为开机自启状态）。

This system doesn't provide enough entropy to quickly generate high-quality random numbers
Installing the rng-utils/rng-tools or haveged packages may help.
On virtualized Linux environments, also consider using virtio-rng.
The service will not start until enough entropy has been collected.

该系统不能提供足够的熵来快速生成高质量的随机数，
安装 rng-utils/rng-tools 或 haveged 软件包可能会有所帮助。
在虚拟化的 Linux 环境中，请考虑使用 virtio-rng。
shadowsocks 服务只有在收集到足够的熵后才会启动。
NoneCopy
这里以 haveged 为例，当然，你也可以选择安装 rng-utils/rng-tools，都是一样的：

# ArchLinux
pacman -S haveged
systemctl enable haveged
systemctl start haveged

# CentOS
yum -y install haveged
## CentOS 6.x
chkconfig haveged on
service haveged start
## CentOS 7.x
systemctl enable haveged
systemctl start haveged
BashCopy
dnsmasq
# CentOS
yum -y install dnsmasq

# ArchLinux
pacman -S dnsmasq
BashCopy
chinadns
# 获取
wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz

# 解压
tar xf chinadns-1.3.2.tar.gz

# 安装
cd chinadns-1.3.2/
./configure
make && make install
BashCopy
dnsforwarder
## 获取 dnsforwarder
git clone https://github.com/holmium/dnsforwarder.git

## 编译 dnsforwarder
cd dnsforwarder/
autoreconf -f -i
./configure --enable-downloader=no
make && make install
BashCopy
如果 make 报错，提示 undefined reference to rpl_malloc，请编辑 config.h.in 文件，把里面的 #undef malloc、#undef realloc 删掉，然后再编译，即：./configure --enable-downloader=no、make && make install。

v2ray
安装很简单，直接使用 v2ray 官方提供的 shell 脚本即可，默认配置开机自启

# 官方安装脚本
bash <(curl -4sSkL https://install.direct/go.sh)

# 安装的文件有
/etc/v2ray/config.json      配置文件
/usr/bin/v2ray/v2ray        V2Ray 程序
/usr/bin/v2ray/v2ctl        V2Ray 工具
/usr/bin/v2ray/geoip.dat    IP 数据文件
/usr/bin/v2ray/geosite.dat  域名数据文件

# 是否安装成功
/usr/bin/v2ray/v2ray -help  # 查看帮助
BashCopy
ss-libev
ArchLinux 建议使用 pacman -S shadowsocks-libev 安装，方便快捷，更新也及时。
CentOS/RHEL 或其它发行版，强烈建议 编译安装，仓库安装的可能会有问题（版本太老或者根本用不了）。
下面的代码完全摘自 ss-libev 官方 README.md，随着时间的推移可能有变化，最好照着最新 README.md 来做。

# Installation of basic build dependencies
## Debian / Ubuntu
sudo apt-get install --no-install-recommends gettext build-essential autoconf libtool libpcre3-dev asciidoc xmlto libev-dev libc-ares-dev automake libmbedtls-dev libsodium-dev
## CentOS / Fedora / RHEL
sudo yum install gettext gcc autoconf libtool automake make asciidoc xmlto c-ares-devel libev-devel
## Arch
sudo pacman -S gettext gcc autoconf libtool automake make asciidoc xmlto c-ares libev

# Installation of Libsodium
export LIBSODIUM_VER=1.0.13
wget https://download.libsodium.org/libsodium/releases/old/libsodium-$LIBSODIUM_VER.tar.gz
tar xvf libsodium-$LIBSODIUM_VER.tar.gz
pushd libsodium-$LIBSODIUM_VER
./configure --prefix=/usr && make
sudo make install
popd
sudo ldconfig

# Installation of MbedTLS
export MBEDTLS_VER=2.6.0
wget https://tls.mbed.org/download/mbedtls-$MBEDTLS_VER-gpl.tgz
tar xvf mbedtls-$MBEDTLS_VER-gpl.tgz
pushd mbedtls-$MBEDTLS_VER
make SHARED=1 CFLAGS=-fPIC
sudo make DESTDIR=/usr install
popd
sudo ldconfig

# Start building
git clone https://github.com/shadowsocks/shadowsocks-libev.git
cd shadowsocks-libev
git submodule update --init --recursive
./autogen.sh && ./configure && make
sudo make install
BashCopy
ssr-libev
shadowsocksr-backup/shadowsocksr-libev（貌似已停止更新，但目前使用没问题，只是一些新特性不支持，如有更好的源请告诉我~）
https://github.com/shadowsocksrr/shadowsocksr-libev/tree/Akkariiin/master，另一个 ssr-libev 源，Akkariiin-* 分支目前仍在更新
本文仍以 shadowsocksr-backup/shadowsocks-libev 为例，毕竟另一个源我没试过，但是这个源我自己用了大半年，没有任何问题，很稳定

# Installation of basic build dependencies
## Debian / Ubuntu
sudo apt-get install --no-install-recommends gettext build-essential autoconf libtool libpcre3-dev asciidoc xmlto libev-dev libc-ares-dev automake libmbedtls-dev libsodium-dev
## CentOS / Fedora / RHEL
sudo yum install gettext gcc autoconf libtool automake make asciidoc xmlto c-ares-devel libev-devel
## Arch
sudo pacman -S gettext gcc autoconf libtool automake make asciidoc xmlto c-ares libev

# Installation of Libsodium
export LIBSODIUM_VER=1.0.13
wget https://download.libsodium.org/libsodium/releases/old/libsodium-$LIBSODIUM_VER.tar.gz
tar xvf libsodium-$LIBSODIUM_VER.tar.gz
pushd libsodium-$LIBSODIUM_VER
./configure --prefix=/usr && make
sudo make install
popd
sudo ldconfig

# Installation of MbedTLS
export MBEDTLS_VER=2.6.0
wget https://tls.mbed.org/download/mbedtls-$MBEDTLS_VER-gpl.tgz
tar xvf mbedtls-$MBEDTLS_VER-gpl.tgz
pushd mbedtls-$MBEDTLS_VER
make SHARED=1 CFLAGS=-fPIC
sudo make DESTDIR=/usr install
popd
sudo ldconfig

# Start building
git clone https://github.com/shadowsocksr-backup/shadowsocksr-libev.git
cd shadowsocksr-libev
./configure --prefix=/usr/local/ssr-libev && make && make install
cd /usr/local/ssr-libev/bin
mv ss-redir ssr-redir
mv ss-local ssr-local
ln -sf ssr-local ssr-tunnel
mv ssr-* /usr/local/bin/
rm -fr /usr/local/ssr-libev
BashCopy
如果编译失败，可以看下这个 issue：https://github.com/shadowsocksrr/shadowsocksr-libev/issues/40（gcc 版本过高导致的）
解决方法就是使用此分支：git clone -b Akkariiin/develop https://github.com/shadowsocksrr/shadowsocksr-libev.git


常见问题
ss-tproxy status 显示不准确（已运行却显示 stopped）
首先检查 ss-tproxy.conf 中的 opts_ss_netstat 选项值是否设置正确，默认是 auto 自选选择模式，即如果有 ss 就优先使用 ss 命令，如果没有才会去使用 netstat 命令；如果自动选择模式对你不适用，那么可以改为 ss 或 netstat 来明确告诉 ss-tproxy 你要使用哪个端口检测命令（有些系统没有 ss，而有些系统却只有 ss）；如果确定不是这个选项的问题，那么我估计你遇到了这个问题：执行 ss-tproxy start|restart 后，发现 pxy/tcp 和 pxy/udp 的状态是 stopped 的，然后再次执行 ss-tproxy status 查看状态，却又发现是 running 的，且代理进程也正常启动了，代理也是正常的；这个其实不是 ss-tproxy 的问题，应该是 ss、netstat 的检测延迟问题，因为第一次状态检测（start|restart）是在 proxy_runcmd 运行之后立即执行的，因为某些原因（具体什么原因我也没详细了解），导致 ss/netstat 没检测到对应的监听端口（或者还有一种情况，就是代理进程此时可能还没监听端口，还在处理其它事，如还在处理命令行选项等等，反正都有可能），然后你再次执行 status 命令查看状态时，因为存在一个时间间隔，所以基本上就能正常检测到了。

ss-tproxy 现在是否支持 ipv6 的透明代理（暂时不支持）
这段时间确实比较忙（996 工作制，苦逼），所以没有时间加上 ipv6 的支持，现在基本上就是给 ss-tproxy 修各种小 bug，大改进估计要段时间。据我的了解，添加 ipv6 支持应该不难，iptables 规则都是差不多的，只不过将 iptables 改为 ip6tables。但是我现在还没有 ipv6 的测试环境（本地宽带还是 ipv4 地址，vps 也还是只有 ipv4），所以也并没有很大的动力去搞这个东西，如果你确实需要支持 ipv6，可以自己加上去，核心的规则部分基本可以照抄，iptables 改为 ip6tables 就行，当然还有其它部分也要进行修改，比如 dns 解析模块那里，目前的实现只考虑了 ipv4（包括各种黑名单等等）。

特别注意，ss-tproxy v3 要求代理服务器支持 udp 转发
就拿 ss/ssr 来说，ss-redir/ssr-redir 需要开启 udp relay 功能，然后 ss-server/ssr-server 也需要开启 udp relay 功能，如果服务器还有防火墙规则，请注意放行对应的 udp 端口，此外还请确认你当地的 ISP 没有对 udp 流量进行恶意丢包。对于 v2ray（vmess），因为它的 udp 是通过 tcp 转发出去的（即 udp over tcp），所以不需要放行服务器的防火墙端口，也不需要担心 ISP 对 udp 流量的恶意干扰，因为根本没有走 udp，而是走的 tcp。如果你因为各种原因无法使用 udp relay，那就需要 tcponly 代理模式了，ss-tproxy v2 版本是支持 tcponly 模式的，但 v3 暂时精简掉了，后期会补上。

如何只对指定的 host/addr 进行代理，其它的通通走直连
其实非常简单，使用 gfwlist 模式即可；gfwlist 模式会读取 gfwlist.txt、gfwlist.ext 两个黑名单文件，如果你只想代理某些域名、IP、网段，其它的都不想代理，可以直接将 gfwlist.txt 文件清空（执行命令 true >/etc/ss-tproxy/gfwlist.txt），然后编辑 gfwlist.ext 文件，填写要代理的域名、IP、网段即可（文件中有格式说明）。注意，在这种模式下就不要执行 update-chnonly、update-gfwlist 命令了，因为它们会操作 gfwlist.txt 文件。

如何将 socks5 代理转换为 ss-tproxy v3 可用的透明代理
如果你因为各种原因无法编译 ss-libev、ssr-libev，但又想使用 ss-tproxy 的透明代理功能，也可以使用 redsocks2 来将任意支持 udp 的 socks5 代理转换为透明代理，这样即使你没有 ss-libev、ssr-libev，也可以使用 ss-tproxy 的透明代理功能（即使用 python 版 ss/ssr + redsocks2 来做透明代理）。当然因为我的系统只安装了 libev 版本，没有安装 python 版本，所以这里依旧使用 ssr-local 作为例子讲解（如果使用 python 版的 ss/ssr，那么你只需要将对应的 ssr-local 替换为 sslocal 命令，但记得开启 udp relay），怎么编译 redsocks2（注意是 redsocks2，不是原版 redsocks）这里就不多说了，官方 readme 有详细说明；我们只需要关注 redsocks2 的配置文件（假设文件路径为 /etc/redsocks2.conf）：

base {
  daemon = on;
  log_info = on;
  reuseport = on;
  redirector = iptables;
  log = "file:/var/log/redsocks2.log";
}

redsocks {
  local_ip = 0.0.0.0;
  local_port = 60080;
  ip = 127.0.0.1;
  port = 61080;
  type = socks5;
}

redudp {
  local_ip = 0.0.0.0;
  local_port = 60080;
  ip = 127.0.0.1;
  port = 61080;
  type = socks5;
}
IniCopy
base 是 redsocks2 的配置，redsocks 是 tcp 透明代理的配置（REDIRECT），redudp 是 udp 透明代理的配置（TPROXY），ip 和 port 是 socks5 服务器的监听地址，在这里就是 ssr-local 的监听地址，而 local_ip、local_port 则是 redsocks2 进程的监听地址，注意是 0.0.0.0:60080/tcp+udp，端口需要和 ss-tproxy.conf 里面的 proxy_tcport、proxy_udport 相同，监听地址也必须为 0.0.0.0。然后就是配置 ss-tproxy.conf，假设 vps 地址为 1.2.3.4：

proxy_tproxy='false'
proxy_server=(1.2.3.4)
proxy_tcport='60080'
proxy_udport='60080'
proxy_runcmd='start_ssrlocal_redsocks2'
proxy_kilcmd='kill -9 $(pidof ssr-local) $(pidof redsocks2)'
BashCopy
然后在 ss-tproxy.conf 的任意位置定义我们的 start_ssrlocal_redsocks2 函数，比如在文件末尾添加：

function start_ssrlocal_redsocks2 {
    (ssr-local -s1.2.3.4 -p8080 -mnone -kpassword -Oorigin -oplain -b0.0.0.0 -l61080 -u </dev/null &>/dev/null &)
    /usr/local/bin/redsocks2 -c /etc/redsocks2.conf
}
BashCopy
iptables: No chain/target/match by that name
如果是 iptables -j TPROXY 这条命令报的错（使用 bash -x /usr/local/bin/ss-tproxy start 查看调试信息），那就是没有 TPROXY 模块。

Syntax error: redirection unexpected
如果运行脚本时报了这个错误，你应该检查一下你的 bash 是不是正常版本，或者使用 ls -l /bin/bash 看下这个文件是否软连接到了其它 shell。

ss-tproxy.conf 中的函数不可重复定义
特别注意，因为 ss-tproxy 和 ss-tproxy.conf 都是一个 bash 脚本，所以这两个文件的内容也必须符合 bash 的语法规则，比如你不能在里面重复定义一个函数，虽然这不会报错，但是只有最后一个函数才会生效，这可能会坑死你，如果你定义了多个同名的 bash 函数，请将它们合并为一个！

BT/PT 问题（global/chnroute 模式）
如果你经常使用 BT/PT 下载，请务必使用 gfwlist 模式，如果使用 global/chnroute 模式，可能会消耗大量的 VPS 流量，甚至可能导致 VPS 被封（因为很多主机商都不允许 BT/PT 下载）。因为使用 iptables 来识别 BT/PT 流量的效率太低（基本都是使用 string 模块来匹配，我个人是无法接受的），所以最好的办法还是使用 gfwlist 模式，因为 gfwlist 模式下只有被墙了的网站才会走代理，其它的都是走直连（绝大多数 BT/PT 流量）。

默认的 8.8.8.8:53 DNS 有问题
请关闭 chinadns 的压缩指针选项，即将 ss-tproxy.conf 中的 chinadns_mutation 改为 false（新版已默认关闭该选项），启用压缩指针时，有时候使用 1.1.1.1、8.8.8.8、8.8.4.4 这些国外 DNS 会有问题，无法正常解析 DNS，导致的现象就是国内网站可以上，但是国外网站不能上。

start 时部分组件 stopped
如果是 dnsmasq 或 chinadns 启动失败，请先检查 /var/log 下面的日志，看看是不是监听地址被占用了（按道理来说，v3 最新版本已经将它们两个的监听端口调的很高了，基本没有与之冲突的监听地址）；如果是 pxy/tcp 或 pxy/udp 启动失败，请检查 ss-tproxy.conf 里面的 proxy_tcport 和 proxy_udport 端口是否与 proxy_runcmd 启动的进程的监听端口一致，因为默认情况下，ss-redir 或 ssr-redir 的监听端口是 1080，而 ss-tproxy 设置的是 60080，当然这个端口是可以随便改的，但是我觉得还是使用高位端口好一些，省得那么多端口冲突。

切换模式后不能正常代理了
从其它模式切换到 gfwlist 模式时可能出现这个问题，原因还是因为内网主机的 DNS 缓存。在访问被墙网站时，比如 www.google.com，客户机首先会进行 DNS 解析，由于存在 DNS 缓存，这个 DNS 解析请求并不会被 ss-tproxy 主机的 dnsmasq 处理（因为根本没从客户机发出来），所以对应 IP 不会添加到 ipset-gfwlist 列表中，导致客户机发给该 IP 的数据包不会被 ss-tproxy 处理，也就是走直连出去了，GFW 当然不会让它通过了，也就出现了连接被重置等问题。解决方法也很简单，对于 Windows，请先关闭浏览器，然后打开 cmd.exe，执行 ipconfig /flushdns 来清空 DNS 缓存，然后重新打开浏览器，应该正常了；对于 Android，可以先打开飞行模式，然后再关闭飞行模式，或许可以清空 DNS 缓存。

有时会无法访问代理服务器
如果你在 ss-tproxy 中使用的是自己的 VPS 的代理服务，那么在除 ss-tproxy 主机外的其他主机上会可能会出现无法访问这台 VPS 的情况（比如 SSH 连不上，但是 ping 没问题），具体表现为连接超时。起初怀疑是 ss-tproxy 主机的 iptables 规则设置不正确，然而使用 TRACE 追踪后却什么都没发现，一切正常；在 VPS 上使用 tcpdump 抓包后，发现一个很奇怪的问题：VPS 在收到来自客户端的 SYN 请求后并没有进行 SYN+ACK 回复，客户端在尝试了几次后就会显示连接超时。于是怀疑是不是 VPS 的问题，谷歌之后才知道，原来是因为两个内核参数设置不正确导致的，这两个内核参数是：net.ipv4.tcp_tw_reuse、net.ipv4.tcp_tw_recycle，将它们都设为 0（也就是禁用）即可解决此问题。其实这两个内核参数默认都是为 0 的，也就是说，只要你没动过 VPS 的内核参数，那么基本不会出现这种诡异的问题。

关于网关的 DHCP 配置细节
这里说的 DHCP 配置仅针对“代理网关”模式，其实如何配置 DHCP 完全取决于你自己的需求，第一种：将路由器 DHCP 分配的 Gateway 和 DNS 改为 ss-tproxy 主机的地址，然后给 ss-tproxy 主机配置静态 IP，注意不能使用 DHCP 来获取，因为获取到的 Gateway 是指向自己的，显然有问题，这种方式下，所有内网主机默认通过 ss-tproxy 代理上网，如果某些主机不想走代理，可以手动将这些主机的 Gateway 和 DNS 改为原来的，就可以恢复直连了，不会走代理。第二种：不改变任何 DHCP 配置，默认情况下除 ss-tproxy 主机本身外，其它主机都是走直连的，如果你想让某些主机走代理上网，可以手动将这些主机的 Gateway 和 DNS 改为 ss-tproxy 主机的 IP，这样就可以走代理上网了。当然如果你的路由器支持为不同的主机设置不同的 Gateway 和 DNS，那么也可以不修改任何内网主机的配置，直接在路由上配置对应的 DHCP 规则就行了。

ss-tproxy 支持什么发行版
一般都支持，没有限制，因为没有使用与特定发行版相关的命令和特性（我自己用的是 ArchLinux），当然我说的是普通 x86、arm 发行版，如果是路由器的那种系统，应该会有点问题，但是移植起来应该不难。我测试过的系统有：ArchLinux、RHEL、CentOS、Debian、Alpine；经过几个网友测试，OpenWrt 貌似也可以，当然需要自己改一些东西，比如 OpenWrt 自带的 bash 是有问题的，阉割版，运行起来会报错。

ss-tproxy 只能运行在网关上吗
显然不是，你可以在一台普通的 linux 主机（甚至是桥接模式下的虚拟机）上运行，并且这种方式也是能够透明代理其它主机的 TCP 和 UDP 的哦。

ss-tproxy 可以运行在副路由上吗
可以。假设你有两个路由器，一主一副，主路由通过 PPPOE 拨号上网，其它设备连接到主路由可以上外网（无科学上网），副路由的 WAN 口连接到主路由的 LAN 口，副路由的 WAN 网卡 IP 可以动态获取，也可以静态分配，此时，副路由自己也是能够上外网的。然后，在副路由上运行 ss-tproxy，此时，副路由已经能够科学上网了，然后，我们在副路由上配置一个 LAN 网段（不要与主路由的 LAN 网段一样），假设主路由的 LAN 网段是 192.168.1.0/24，副路由的 LAN 网段是 10.10.10.0/24。然后指定一个网关 IP 给副路由的 LAN 口，假设为 10.10.10.1，开启副路由上的 DHCP，分配的地址范围为 10.10.10.100-200，分配的网关地址为 10.10.10.1，分配的 DNS 服务器为 10.10.10.1。现在，修改 ss-tproxy.conf 的内网网段为 10.10.10.0/24，重启 ss-tproxy，然后连接到副路由的设备应该是能够科学上网的。你可能会问，为什么不直接在主路由上安装 ss-tproxy 呢？假设这里的副路由是一个树莓派，那么我不管在什么网络下（公司、酒店），只要将树莓派插上，然后我的设备（手机、笔记本）只需要连接树莓派就能无缝上网了，同时又不会影响内网中的其它用户，一举两得（当然我只是举个栗子，实际上可能没想的这么方便）。

ss-tproxy 可以运行在内网主机上吗（代理网关）
可以。先解释一下这里的“代理网关”（不知道叫什么好），由网友 @feiyu 启发。他的意思是，将 ss-tproxy 部署在一台普通的内网主机上（该主机的网络配置不变），然后将其他内网主机的网关和 DNS 指向这台部署了 ss-tproxy 的主机，进行代理。方案是可行的，我在 VMware 环境中测试通过。注意，这个“代理网关”可以是一台真机，也可以是一台虚拟机（桥接模式），比如在 Windows 系统中运行一个 VMware 或 VirtualBox 的 Linux 虚拟机，在这个虚拟机上跑 ss-tproxy 当然也是可以的（你还别说，真有不少人是这样用的）。

切换模式、切换节点不方便，能否避免直接修改文件
这确实是个问题，切换节点需要修改配置文件，切换模式需要修改配置文件，有没有更加简便一些的方式？抱歉，没有。不过因为 ss-tproxy.conf 是一个 shell 脚本，所以我们可以在这里面做些文章。如果你有很多个节点（付费机场一般都是，当然 ss-tproxy 可能更适合自建代理的用户），可以这样做：

# 编辑 /etc/ss-tproxy/ss-tproxy.conf 配置文件，在尾部添加两行：
[ "$2" ] && ln -sf /etc/ss-tproxy/$2.conf /etc/ss-tproxy/node.conf
source /etc/ss-tproxy/node.conf

# 在 /etc/ss-tproxy 下创建节点文件，如 node-1.conf、node-2.conf
cd /etc/ss-tproxy
touch node-1.conf node-2.conf

# 编辑 node-1.conf、node-2.conf，每个文件都代表一个不同的节点
# 以 node-1.conf 为例，使用 gfwlist 模式，服务器为 ss-node-1
mode='gfwlist'
proxy_tproxy='false'
proxy_server=(ss-node-1)
proxy_tcport='60080'
proxy_udport='60080'
proxy_runcmd='systemctl start ss-redir@node1'
proxy_kilcmd='systemctl stop ss-redir@node1'

# node-N.conf 中可以包含 ss-tproxy.conf 中的所有变量，如 node2
mode='chnroute'
proxy_tproxy='false'
proxy_server=(ss-node-2)
proxy_tcport='60080'
proxy_udport='60080'
proxy_runcmd='systemctl start ss-redir@node2'
proxy_kilcmd='systemctl stop ss-redir@node2'
dns_remote='8.8.4.4:53'

# 语法：ss-tproxy start|stop|restart [node-1|node-2|node-N]
# 说明：第 2 个参数其实就是节点文件的名称（注意没有 .conf 后缀名）
ss-tproxy start|stop|restart node-1 # 切换至 /etc/ss-tproxy/node-1.conf
ss-tproxy start|stop|restart node-2 # 切换至 /etc/ss-tproxy/node-2.conf
# 另外注意，这个节点名称的参数是可选的，默认就是上次切换后的那个节点，举个栗子：
ss-tproxy restart node-1 # 切换至 /etc/ss-tproxy/node-1.conf（内部会执行 ln -sf 来切换）
ss-tproxy restart        # 依旧使用 node-1.conf 节点，因为 node.conf 仍然指向 node-1.conf
ss-tproxy restart node-2 # 切换至 /etc/ss-tproxy/node-2.conf（内部会执行 ln -sf 来切换）
ss-tproxy restart        # 依旧使用 node-2.conf 节点，因为 node.conf 仍然指向 node-2.conf
BashCopy
配置文件以及其它文件的路径是否可更改
当然是可以的，比如你想将 /etc/ss-tproxy 默认目录改为 /opt/ss-tproxy，只需要修改两个地方，一个是 ss-tproxy 脚本的第 3 行，将 main_conf 变量改为 /opt/ss-tproxy/ss-tproxy.conf；另一个是 /opt/ss-tproxy/ss-tproxy.conf 里面的 file_* 变量，改为 /opt/ss-tproxy 目录下的就行了。顺便说一句，ss-tproxy 脚本本身也并不是说一定要放到 /usr/local/bin 目录下，只是我个人喜欢将第三方命令放到这个目录而已。

CentOS 7.x 必须关闭 firewalld 防火墙
当然 RHEL 7.x 也一样，因为 iptables 和 firewalld 都是 netfilter 的用户空间配置工具，而默认情况下，RHEL/CentOS 7.x 会将 firewalld 服务设置为开机自启动，而且还会设置一些防火墙规则，如果你不关闭这个服务，可能在做代理网关时，会遇到无法上网的情况，所以请务必执行 systemctl disable firewalld 来关闭它（然后重启系统）。当然也不是说一定要关闭 firewalld，如果你对 iptables 和 firewalld 比较熟悉，完全可以自由支配它们。

服务器的 IP 或域名经常变动，每次都要改 IP，怎么办
基本上我可以肯定的说，ss-tproxy 不适合你，用 ss-tproxy 脚本的人大多数也是用的自己 VPS 上的代理服务器。

为什么必须使用 bash，而不能使用 sh、zsh 这些 shell
因为 ss-tproxy 脚本里面用到了非常多的 bash 高级重定向，而且有些 built-in 命令和 sh、zsh 的用法不同，无法兼容。

为什么使用 ipip.net 的 chnroute 源，而不是 APNIC 的源
这个怎么说呢，ipip.net 的源我觉得挺好的，用了很久也没啥问题，当然如果需要，你也可以自己改为其它的 chnroute 源，比如改为 APNIC 的大陆地址段列表。但是 ss-tproxy.conf 中并未提供这个选项，该怎么改呢？其实不难，你不需要去修改脚本，只需要在 ss-tproxy.conf 文件末尾添加以下内容（基本框架可以照抄，具体的更新命令可以自定义）：

if [ "$1" = 'update-chnroute' ]; then

    # 自定义的更新命令 - begin
    curl -4sSkL 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F'|' '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' >$file_chnroute_txt
    # 自定义的更新命令 - end

    echo '-N chnroute hash:net' >$file_chnroute_set; sed -r 's/^.+$/-A chnroute &/' $file_chnroute_txt >>$file_chnroute_set
    reserved_address=('0.0.0.0/8' '10.0.0.0/8' '127.0.0.0/8' '169.254.0.0/16' '172.16.0.0/12' '192.168.0.0/16' '224.0.0.0/4' '240.0.0.0/4')
    for addr in "${reserved_address[@]}"; do echo "$addr" >>$file_chnroute_txt; done
    exit 0
fi
BashCopy
其实你可以完全模仿这个例子，来重写 ss-tproxy 中定义的全部函数，并且还可以添加额外的 ss-tproxy COMMAND，就看你怎么写了。

如何提高代理的性能（当然仅针对 ss-redir、ssr-redir 这些）
首先，优先选择 C 语言版的 SS/SSR，当然 Python 版也没问题，然后服务器的监听端口我个人觉得 80 和 443 比较好一点，8080 也可以，貌似这些常用端口可以减少 QoS 的影响，当然这只是我个人的一些意见，然后就是修改内核参数，ss-tproxy 主机的 sysctl.conf 以及 vps 的 sysctl.conf 都建议修改一下，最好同步设置。另外我还有一个提速小技巧，那就是在 ss-tproxy 中启动多个 ss-redir、ssr-redir，它们都监听同一个地址和端口（是的你没听错），对于 ss-redir，需要添加一个 --reuse-port 选项来启用端口重用，而对于 ssr-redir，默认情况下就启用了端口重用，而且也没有这个选项。为了简单，这里就以 ss-redir 为例，老规矩，修改 ss-tproxy.conf，添加一个函数，用来启动 N 个 ss-redir 进程：

function start_multiple_ssredir {
    ss_addr='1.2.3.4'
    ss_port='8443'
    ss_method='aes-128-gcm'
    ss_passwd='passwd.for.test'
    (ss-redir -s$ss_addr -p$ss_port -m$ss_method -k$ss_passwd -b0.0.0.0 -l60080 -u --reuse-port </dev/null &>/dev/null &)
    for ((i = 1; i < $1; ++i)); do
        (ss-redir -s$ss_addr -p$ss_port -m$ss_method -k$ss_passwd -b0.0.0.0 -l60080 --reuse-port </dev/null &>/dev/null &)
    done
}
BashCopy
然后修改我们的启动命令，即 proxy_runcmd，将它改为 proxy_runcmd='start_multiple_ssredir 4'，其中 4 可以改为任意正整数，这个数值的意思是启动多少个 ss-redir 进程，一般建议最多启动 CPU 核心数个 ss-redir 进程，太多了性能反而会下降，当然你也可以将它改为 1，此时只会启动一个进程，也就没有所谓的加速了（多个 ss-redir 进程的加速仅针对多线程下载、多终端并发访问的情况，话虽如此，但是效果还是很明显的）。

ss-libev、ssr-libev 如何进行多节点负载均衡（SO_REUSEPORT）
其实非常简单，和上面的多进程 ss-libev、ssr-libev 加速差不多，只不过 ss_addr 不同而已，反正只要这些 ss-redir、ssr-redir 进程监听同一个 addr:port（127.0.0.1:60080），就可以正常使用，实际上 ss-tproxy 并不关心你使用的是哪个 proxy_server，也不关心你使用的是 ss-libev 还是 ssr-libev 还是 v2ray。有必要声明是，通过 SO_REUSEPORT 端口重用实现的负载均衡是“平均分配”的，假设有 4 个进程同时监听 127.0.0.1:60080，那么内核会将客户端连接平均分配给这 4 个进程（每个进程分配到的概率为 25%）。例子：假设我有 2 个 ss 服务器（1.1.1.1、2.2.2.2），2 个 ssr 服务器（3.3.3.3、4.4.4.4），我想同时使用这 4 个服务器（负载均衡），该如何做？

proxy_server=(1.1.1.1 2.2.2.2 3.3.3.3 4.4.4.4)
proxy_runcmd='start_load_balancing'

function start_load_balancing {
    (ss-redir  -s1.1.1.1 -p80  -maes-128-gcm -kpassword -b0.0.0.0 -l60080 -u --reuse-port </dev/null &>/dev/null &)
    (ss-redir  -s2.2.2.2 -p80  -maes-128-gcm -kpassword -b0.0.0.0 -l60080 -u --reuse-port </dev/null &>/dev/null &)
    (ssr-redir -s3.3.3.3 -p443 -maes-128-cfb -kpassword -b0.0.0.0 -l60080 -u              </dev/null &>/dev/null &)
    (ssr-redir -s4.4.4.4 -p443 -maes-128-cfb -kpassword -b0.0.0.0 -l60080 -u              </dev/null &>/dev/null &)
}
BashCopy
使用“代理网关”模式时，ss-tproxy stop 后，内网主机无法上网
如果 ss-tproxy 主机上没有运行 DNS 服务器（注意，如果代理网关上已经有一个运行在 53 端口上的 dns 服务器，就不要再执行这里的操作了，有些人看都不没看清就直接照抄过去），那就会出现这个问题，因为你的 DNS 设为了 ss-tproxy 主机的 IP，而你执行 stop 操作后，ss-tproxy 上的 dnsmasq 就会被 kill（注意 kill 的是 ss-tproxy 运行的那个 dnsmasq 进程，系统运行的或者你自己运行的 dnsmasq 进程不会被 kill），使得内网主机无法解析 DNS，从而无法上网。解决方法也很简单，修改 ss-tproxy.conf，添加两个钩子函数，post_stop 钩子函数的作用是在 stop 之后启动一个 dnsmasq，监听 0.0.0.0:53 地址，给内网主机提供普通的 DNS 服务，pre_start 钩子函数的作用是在 start 之前 kill 这个 53 端口的 dnsmasq，因为 ss-tproxy 代理启动后，这个 dnsmasq 进程就没有存在的必要了。

function post_stop {
    dnsmasq --conf-file=/dev/null --no-resolv --server=${dns_direct} &>/dev/null
}

function pre_start {
    if [ "$opts_ss_netstat" != "netstat" -a "$(command -v ss &>/dev/null && echo 'true')" ]; then
        ss -lnpu | awk -F, '/:53[ \t].+"dnsmasq"/ {print $(NF-1)}' | awk -F= '{print $2}' | sort | uniq | xargs kill -9 &>/dev/null
    else
        netstat -lnpu | awk -F/ '/:53[ \t].+dnsmasq/ {print $(NF-1)}' | awk '{print $NF}' | sort | uniq | xargs kill -9 &>/dev/null
    fi
}
BashCopy
为什么使用 ping 命令来解析域名？而不是 dig、nslookup、getent
起初我是使用 getent hosts $domain_name 来解析域名的，但是后来我发现在某些系统上没有 getent 命令，所以我就改为了 ping 来解析。

为什么不支持自定义 dnsmasq 和 chinadns 的端口，只能使用默认的
因为没有这个必要，默认情况下，dnsmasq 监听 60053 端口，chinadns 监听 65353 端口，基本上没哪个进程会监听这两个高位端口，我之所以设置为高位端口也是为了尽可能避免端口冲突问题，在早期版本中，dnsmasq 是监听在 53 端口的，但是我收到了很多关于 dnsmasq 监听端口冲突的反馈，虽然端口冲突问题不难解决，但是我为了一劳永逸，直接将这个端口改为了 60053，从根本上避免了这个问题。

为什么 gfwlist 模式需要用到 perl？而不是使用 sed、awk、grep 这些
因为 Perl 的正则表达式真的很强大，而且没有所谓的兼容性问题，sed、awk、grep 基本上都有兼容性问题，太烦了。

支持 ss-libev 或 ssr-libev 与 kcptun 一起使用吗
可以，当然需要进行一些特殊改造。ss-libev + kcptun 和 ssr-libev + kcptun 操作起来都差不多，为了简单，这里就以 ss-libev 为例。首先我假设你已经在 vps 上运行了 ss-server 和 kcptun-server，并且 ss-server 监听 0.0.0.0:8053/tcp+udp（监听 0.0.0.0 是为了处理 ss-redir 的 udp relay，因为 kcptun 只支持 tcp 协议的加速），kcptun-server 监听 0.0.0.0:8080/udp（用来加速 ss-redir 的 tcp relay，会被封装为 kcp 协议，使用 udp 传输）；当然这些端口都是可以自定义的，我并没有规定一定要使用这两个端口！然后编辑 ss-tproxy.conf，修改这几条配置（假设 vps 地址为 1.2.3.4）：

proxy_server=(1.2.3.4)
proxy_runcmd='start_sslibev_kcptun'
proxy_kilcmd='kill -9 $(pidof ss-redir) $(pidof kcptun-client)'
BashCopy
注意，kcptun 的 client 和 server 程序默认并不是叫做 kcptun-client、kcptun-server，而是一个很难听的名字，如 client_linux_amd64、server_linux_amd64，我为了好记，将它改为了 kcptun-client、kcptun-server；如果你没有改这个名字，那么你就需要修改一下上面的 kcptun-client 为对应的 client 二进制文件名，但是我建议你改一下，可以省去不少麻烦。然后你可能注意到了 start_sslibev_kcptun 这个命令，这实际上是我们待会要定义的一个函数，方便启动 ss-redir 和 kcptun-client，而不用在 proxy_runcmd 中写很长的启动命令；然后，在 ss-tproxy.conf 的任意位置添加以下配置（我个人喜欢在文件末尾添加），假设 method 为 aes-128-gcm，password 为 passwd.for.test：

function start_sslibev_kcptun {
    (kcptun-client -l :8080 -r 1.2.3.4:8080 --mode fast2 </dev/null &>/dev/null &)
    (ss-redir -s127.0.0.1 -p8080 -maes-128-gcm -kpasswd.for.test -b0.0.0.0 -l$proxy_tcport    </dev/null &>/dev/null &)
    (ss-redir -s1.2.3.4   -p8053 -maes-128-gcm -kpasswd.for.test -b0.0.0.0 -l$proxy_udport -U </dev/null &>/dev/null &)
}
BashCopy
为什么 REDIRECT + TPROXY 模式的代理需要监听 0.0.0.0
我在 README 里面强调过，ss-redir 和 ssr-redir 的监听地址需要指定为 0.0.0.0（当然其它代理软件也一样，如果是 REDIRECT + TPROXY 组合方式的话），如果你不指定这个监听地址，或者指定为监听 127.0.0.1，那么你会发现内网主机是无法正常代理上网的，为什么呢？据我个人猜测，应该是 iptables 的 REDIRECT 的问题（听说改为 DNAT 可以避免这个问题，但经过验证貌似也是有问题的），具体什么原因我也不想深究了，没多大意义。

如何在 ss-tproxy 中集成 koolproxy 等广告过滤软件
adbyby 和 koolproxy 是路由固件中常见的两个广告过滤插件，因为 adbyby 不支持 https 过滤，所以这里就以 koolproxy 作为例子讲解。首先打开 https://firmware.koolshare.cn/binary/KoolProxy/，下载对应平台的 koolproxy 二进制文件，然后将它放到 /opt/koolproxy 目录（当然放哪里没要求），并命名为 koolproxy，然后加上可执行权限；进入 /opt/koolproxy 目录，执行 ./koolproxy --cert 生成 koolproxy HTTPS 证书等文件；因为待会我们需要用一个非 root 用户来运行 koolproxy 进程（可以从 /etc/passwd 中随便找一个已存在的用户，比如 daemon），所以我们要先将 /opt/koolproxy 目录的所有者改为 daemon，不然 koolproxy 自动更新广告过滤规则时会遇到权限问题，即执行命令 chown -R daemon:daemon /opt/koolproxy；最后编辑 ss-tproxy.conf，在文件末尾添加这两个钩子函数，就可以实现 koolproxy + ss-tproxy 组合方式的 广告过滤 + 透明代理（广告过滤或多或少都会影响性能，如果设备性能不太好，不建议集成 koolproxy 等广告过滤软件，这种情况下在客户端进行广告过滤会好一些）：

function post_start {
    su -s/bin/sh -c'/opt/koolproxy/koolproxy -d -l2 -p65080 -b/opt/koolproxy/data' daemon
    if [ "$proxy_tproxy" = 'true' ]; then
        iptables -t mangle -I SSTP_OUT -m owner ! --uid-owner daemon -p tcp -m multiport --dports 80,443 -j RETURN
        iptables -t nat    -I SSTP_OUT -m owner ! --uid-owner daemon -p tcp -m multiport --dports 80,443 -j REDIRECT --to-ports 65080
        for intranet in "${ipts_intranet[@]}"; do
            iptables -t mangle -I SSTP_PRE -m mark ! --mark $ipts_rt_mark -p tcp -m multiport --dports 80,443 -s $intranet ! -d $intranet -j RETURN
            iptables -t nat    -I SSTP_PRE -m mark ! --mark $ipts_rt_mark -p tcp -m multiport --dports 80,443 -s $intranet ! -d $intranet -j REDIRECT --to-ports 65080
        done
    else
        iptables -t nat -I SSTP_OUT -m owner ! --uid-owner daemon -p tcp -m multiport --dports 80,443 -j REDIRECT --to-ports 65080
        for intranet in "${ipts_intranet[@]}"; do
            iptables -t nat -I SSTP_PRE -s $intranet ! -d $intranet -p tcp -m multiport --dports 80,443 -j REDIRECT --to-ports 65080
        done
    fi
}

function post_stop {
    kill -9 $(pidof koolproxy) &>/dev/null
}
[ss-tproxy 常见问题解答](https://www.zfl9.com/ss-redir.html#%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)
