# Tunnix
昨天（9 月 16 日）Perseverance 询问在校外打靶场 PWN 题目的方法。他已经获知了在校外打 Web 题的方法：通过校 VPN 授权后，使用 `https://172-26-92-251-<port>-p.ivpn.hitwh.cn` 连接到靶机。（注意如果不是在浏览器里的话需要复制 Cookies 过去）

在昨天的尝试中，我们用 ttyd 搭建了一个 WebShell。然而它依赖 WebSocket，但校园 VPN 似乎并不支持 WebSocket。

杨哲思遂转变思路，直接搭建通道题目，用 HTTP 去承载 TCP 请求，使校园 VPN 能够通过。

经过询问蓝色大肥鱼老师，杨哲思找到了宝藏项目 [Tunnix](https://github.com/aeroxy/tunnix)。它通过纯 HTTP(SSE) 进行 SOCKS/HTTP 代理，对于我们的场景堪称完美。它还是个 Rust 项目，不太容易出什么奇奇怪怪的二进制漏洞（雾），而且才搓出来几个月，维护还很活跃。

题目内服务：
```
11451:   Tunnix 服务端
1145:    隐藏的 Flag 服务
80:
    /       -> index.html
    /tunnix -> 转发到 11451 端口
（然后 GZCTF 平台又把 80 映射到 32xxx 端口）
```


## 教程
直接从 GitHub Releases 拿到二进制文件。（上面只有 Linux x86_64 和 MacOS arm64，别的平台自己 Cargo 编译）我懒得编译，直接放 WSL（Ubuntu）里面跑。

GitHub 连不上优先考虑用 Watt Toolkit（Steam++）反代。永远不要优先考虑魔法上网。

### 在校内
链路示意：

```
选手程序
    -(SOCKS5)-> 选手的 localhost:7890（由 Tunnix 客户端监听的代理）
    -(HTTP)-> 172.26.92.251:<端口>（容器中的 Tunnix 服务端）
    -(TCP)-> 172.17.0.1:<端口>（此 IP 为相对于容器，靶场中的 PWN 题目）
    或 
    -(HTTP)-> 127.0.0.1:1145（此 IP 为相对于容器，Tunnix 同容器中的发 Flag 服务）
```

建个配置文件：
```toml
[client]
password = "114514" # 我在那个题目里面设的密码
local_addr = "127.0.0.1:7890" # 这是默认的本地代理位置
server_url = "http://172.26.92.251:<端口>/tunnix"
```

在确认已经启动题目后，运行：
```bash
tunnix client --config="<配置文件路径>"
```

看到如下输出代表准备完毕：
```
2026-09-17T11:37:08.082469Z  INFO SSE stream connected
2026-09-17T11:37:08.097228Z  INFO Tunnel established
2026-09-17T11:37:08.097298Z  INFO Proxy listening on 127.0.0.1:7890 (SOCKS5 + HTTP)
```

然后再开了个同 WSL 里面的 shell，使用 curl 或 ncat 连接目标题目。

cURL 访问 Tunnix 题目中隐藏的 `1145` 端口：
```bash
curl -v --socks5-hostname 127.0.0.1:7890 http://127.0.0.1:1145 # 这里，我们写的 127.0.0.1:1145 是站在 Tunnix 服务端的视角看的
# 其实这个不一定要用SOCKS5，因为HTTP本身就能同时表达代理服务器和目标服务器，因此下面这个也可以：
curl -v --proxy http://127.0.0.1:7890 http://127.0.0.1:1145
```

Ncat 访问另一个 PWN 题目：
```bash
sudo apt update && sudo apt install ncat # 一般的发行版可能没有预装 ncat
ncat --proxy 127.0.0.1:7890 --proxy-type socks5 172.17.0.1 <题目端口> # 172.17.0.1 代表 Docker 的宿主机
```

PwnTools 本身也支持 SOCKS 代理，但我没试过，朋友们可以试试看。
```py
from pwn import *
context.proxy = (socks.SOCKS5, "127.0.0.1", 7890)

# 后面写攻击逻辑
```
```
python3 exp.py
```

可以用 proxychains 访问，但是这种方法经过测试仅在校内有效，校外会出现问题。

### 在校外（通过 VPN）
链路示意：

```
选手程序
    -> 选手的 localhost:7890（由 Tunnix 客户端监听的代理）
    -> https://172-26-92-251-<port>-p.ivpn.hitwh.cn（校 VPN）
    -> 172.26.92.251:<端口>（容器中的 Tunnix 服务端）
    -> 172.17.0.1:<端口>（此 IP 为相对于容器，靶场中的 PWN 题目）
    或 127.0.0.1:1145（此 IP 为相对于容器，Tunnix 同容器中的发 Flag 服务）
```

内容都是相似的，唯一的不同是需要 Cookie。

先访问 `ivpn.hitwh.edu.cn` 通过认证，然后按照 Perseverance 的方法访问 Tunnix 题目，在 DevTools 的 Network 栏内找到 Cookie 的值，大概长这样：

```
TWFID=**********
```
然后配置文件跟上面差不多，但是要改掉 `server_url`，然后把 Cookie 塞进去：

```toml
[client]
password = "114514" # 我在那个题目里面设的密码
local_addr = "127.0.0.1:7890" # 这是默认的本地代理位置
server_url = "https://172-26-92-251-<端口>-p.ivpn.hitwh.edu.cn/tunnix"

[client.headers]
Cookie = "TWFID=..."
```

再用 `tunnix client --config="<配置文件路径>"` 启动，然后用 curl 或 ncat 访问目标题目。

