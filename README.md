# VSCode + WSL 里配 Claude Code 连不上,大概率是 WSL 的 DNS 问题

在 **Windows + WSL2 + VSCode Remote WSL** 环境里第一次配 Claude Code,连不上十有八九不是 Claude Code 本身的毛病,而是 WSL 的网络 / DNS 没配好,访问不了 `api.anthropic.com`。

下面是一次完整的排查记录。

## 一个容易忽略的前提

Claude Code 跑在你当前的开发环境里,用的就是那个环境的网络:

- VSCode 连本地 Windows → 走 Windows 网络
- VSCode 连 WSL → 走 WSL 网络
- VSCode 连远程服务器 → 走远程网络

所以**浏览器能打开网页,不代表 Claude Code 能连上**——浏览器在 Windows,Claude Code 可能在 WSL,两套网络互不相干。排查一律以 Claude Code 的实际运行环境为准。

Remote WSL 场景下,先在 WSL 终端确认 CLI 装好了、人也确实在 WSL 里:

```bash
which claude        # 看到 /home/xxx/.npm-global/bin/claude 之类的路径即可
pwd; hostname       # 路径是 /home/xxx/... 说明在 WSL 中
```

## 一条命令定位问题

在 WSL 终端直接打:

```bash
curl -I https://api.anthropic.com
```

对照返回结果:

| 返回 | 含义 |
|------|------|
| `HTTP/2 404` + `server: cloudflare` | **通了**,根路径本来就没页面,正常 |
| `Could not resolve host` | DNS 解析失败(我这次的根因) |
| `Failed to connect` | 连接失败,网络 / 路由 / 代理问题 |
| `Resolving timed out` | DNS 服务器没响应 |

我这次是 `Could not resolve host`,顺着往下查。

## 分层排查

```bash
ping -c 3 8.8.8.8                      # 1. IP 出口通不通
cat /etc/resolv.conf                   # 2. DNS 指向哪
getent ahostsv4 api.anthropic.com      # 3. 域名能不能解析
curl -4 -I --connect-timeout 8 https://api.anthropic.com   # 4. 强制 IPv4 再试
```

第 4 步加 `-4` 是为了避免 IPv6 解析成功、但路由不可用导致的误判。

我这边 `ping` 是通的,但 `/etc/resolv.conf` 里的 DNS 指向了一个不可用地址,域名自然解析不了。**根因锁定:WSL 的 DNS 配置坏了**,跟 Claude Code 无关。

## 解决:NAT 模式 + 固定 DNS

改三个文件即可。

**1. Windows 侧 `C:\Users\<用户名>\.wslconfig`**

```ini
[wsl2]
networkingMode=nat
dnsTunneling=false
autoProxy=false
```

> 如果你顺手写了 `processors`,别超过机器实际逻辑核数,否则 WSL 会报错、配置不生效。

**2. WSL 侧 `/etc/wsl.conf`**——关掉自动生成 DNS:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true

[user]
default=<你的WSL用户名>

[network]
generateResolvConf=false
EOF
```

**3. WSL 侧 `/etc/resolv.conf`**——写死可用 DNS:

```bash
sudo rm -f /etc/resolv.conf
sudo tee /etc/resolv.conf >/dev/null <<'EOF'
nameserver 1.1.1.1
nameserver 8.8.8.8
nameserver 9.9.9.9
options timeout:2 attempts:1 single-request-reopen
EOF
```

改完回到 **Windows 的 CMD / PowerShell** 重启 WSL:

```cmd
wsl --shutdown
wsl
```

## 验证

重进 WSL 后,只要这两条对了就说明链路通了:

```bash
getent ahostsv4 api.anthropic.com
# → 160.79.104.10 STREAM api.anthropic.com

curl -4 -I --connect-timeout 8 https://api.anthropic.com
# → HTTP/2 404 / server: cloudflare
```

然后关掉 VSCode,在 Windows 里 `wsl --shutdown`,重新打开并连回 WSL 项目,再启动 Claude Code 即可。

## 几个容易踩的坑

**`127.0.0.1` 在 WSL 里指的是 WSL 自己,不是 Windows 宿主机。** 如果代理软件跑在 Windows 上,WSL 里 `curl --proxy http://127.0.0.1:7890` 是连不上的。我一开始就卡在这——而且后来发现 Windows 那个 7890 端口压根不是个可用的 HTTP Proxy,围着它折腾纯属浪费时间。**先确认 WSL 能不能裸连 API,别一上来就配代理。**

**别让残留的代理环境变量背锅。** 启动前查一下,有错的就清掉:

```bash
env | grep -i proxy
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
```

**`wsl --shutdown` 是 Windows 命令**,在 Ubuntu 里执行会提示 `Command 'wsl' not found`,这是正常的,别去 WSL 里装它。

**API 返回 404 不是失败。** 根路径没页面而已,只要能解析域名、建起 HTTPS、收到响应,就算通。

## 排查顺序速查

```text
claude 命令是否存在
  → 确认 Claude Code 实际运行环境(WSL?Windows?)
  → curl 测 api.anthropic.com
  → 查 IP 出口(ping)
  → 查 DNS(resolv.conf / getent)
  → 查 IPv4/IPv6
  → 查有没有误配代理
  → 最后才轮到插件登录 / API Key
```

** 这次的解法就是 WSL 改 NAT 模式 + 写死 DNS + 别瞎配代理。
