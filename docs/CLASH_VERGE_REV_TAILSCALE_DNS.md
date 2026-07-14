# Clash Verge Rev + Tailscale DNS 配置说明

## 目标

这套配置同时满足以下需求：

- Clash Verge Rev/Mihomo 接管普通公网 DNS，并继续使用 Fake-IP 分流。
- 只有当前 Tailnet 的完整域名 `*.tailxxxxxx.ts.net` 交给 Tailscale MagicDNS。
- Tailscale subnet routes 继续生效。
- 不让 Tailscale 接管全部系统 DNS，避免公共域名回落到本地 `114.114.114.114`。
- 不依赖动态变化的 macOS `utunN` 接口编号。
- 不修改机场远程订阅，只通过 Clash Verge Rev 的本地配置和 macOS split DNS 补充行为。

## 最终架构

```text
普通公网域名，例如 www.google.com
  -> macOS 默认 DNS（当前显示为 114.114.114.114）
  -> Mihomo TUN dns-hijack
  -> Mihomo Fake-IP / 规则分流 / 代理节点

完整 Tailnet 域名，例如 device.tailxxxxxx.ts.net
  -> macOS Super DNS
  -> /etc/resolver/tailxxxxxx.ts.net
  -> 100.100.100.100（Tailscale Quad100 / MagicDNS）
  -> 返回真实 Tailscale IP 100.x.x.x
  -> Tailscale utun 接口
```

两个 DNS 系统的职责是分开的：

| 查询 | 负责者 | 预期结果 |
| --- | --- | --- |
| `www.google.com` | Mihomo | `198.18.x.x` Fake-IP |
| `*.tailxxxxxx.ts.net` | macOS split DNS + Quad100 | 真实 `100.x.x.x` |
| Tailnet 短名称，如 `device` | 当前未配置 | 默认不能解析，使用完整域名 |

## Tailscale 配置

关闭本机的 Tailscale DNS 接管：

```bash
tailscale set --accept-dns=false
```

这只关闭 Tailscale 对系统 DNS 的接管，不会关闭：

- Tailscale 网络连接；
- MagicDNS 在 `100.100.100.100` 上的应答；
- subnet routes；
- 对 `100.64.0.0/10` 的访问。

当前验证状态：

```text
Tailscale DNS: disabled
MagicDNS: enabled tailnet-wide
Tailnet suffix: tailxxxxxx.ts.net
```

## macOS split DNS 配置

创建 `/etc/resolver/tailxxxxxx.ts.net`：

```text
nameserver 100.100.100.100
port 53
```

配置命令：

```bash
sudo mkdir -p /etc/resolver
sudo tee /etc/resolver/tailxxxxxx.ts.net >/dev/null <<'EOF'
nameserver 100.100.100.100
port 53
EOF

sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

macOS 的 Super DNS 会按照最长域名后缀选择 resolver：

- `*.tailxxxxxx.ts.net` 命中 `/etc/resolver/tailxxxxxx.ts.net`；
- 其他域名继续使用系统默认 resolver，并被 Mihomo TUN 劫持。

可以用下面的命令确认配置已经加载：

```bash
scutil --dns
```

预期包含：

```text
resolver
  domain   : tailxxxxxx.ts.net
  nameserver[0] : 100.100.100.100
  port     : 53
```

## Clash Verge Rev 当前运行配置

最终生成文件位于：

```text
~/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml
```

不要直接编辑这个生成文件；持久修改应写入当前订阅关联的 Merge/Script 配置。

当前关键配置如下：

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  respect-rules: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.0/16
  fake-ip-filter-mode: whitelist

tun:
  enable: true
  stack: mixed
  device: utunN
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true
  strict-route: true
  route-exclude-address:
    - 10.0.0.0/8
    - 100.64.0.0/10
    - 127.0.0.0/8
    - 169.254.0.0/16
    - 172.16.0.0/12
    - 192.168.0.0/16
    - FC00::/7
    - FE80::/10
    - ::1/128
  dns-hijack:
    - any:53
    - tcp://any:53
```

`100.64.0.0/10` 必须保留在 `route-exclude-address` 中。这样真实的 Tailscale 地址和 Quad100 不会被送回 Clash TUN，而是按系统路由进入 Tailscale 接口。

## 关于当前 `nameserver-policy` 的冗余项

当前生成配置中仍能看到：

```yaml
dns:
  nameserver-policy:
    "+.tailxxxxxx.ts.net": 100.100.100.100
    geosite:private: system
```

这条规则已经不是系统访问 MagicDNS 的实际路径，可以从 Merge 中删除。

原因是 macOS split DNS 会在查询进入 Mihomo 之前，就把 `*.tailxxxxxx.ts.net` 直接发给 Quad100。正常应用不需要再让 Mihomo 转发一次。

保留它不会影响 macOS split DNS，但直接查询 Mihomo 时可能超时：

```bash
dig device.tailxxxxxx.ts.net @127.0.0.1 -p 1053
```

已抓到的 Mihomo 日志表明：

```text
resolve device.tailxxxxxx.ts.net A from udp://100.100.100.100:53
Auto detect interface for 100.100.100.100 -> enN
all DNS requests failed: i/o timeout
```

Quad100 实际位于 Tailscale 的 `utunN` 接口，而 Mihomo 的自动出口选择了物理接口 `enN`，所以这条“由 Mihomo 再转发给 Quad100”的路径会超时。

不建议把 `#utun10` 写死到 DNS 地址中，因为 macOS 的 `utunN` 编号可能在重启或重新连接后变化。使用 `/etc/resolver` 可以直接交给系统路由处理，避免依赖接口编号。

## 为什么 Tailnet 域名不再返回 Fake-IP

当前 `fake-ip-filter-mode` 是 `whitelist`：只有命中 `fake-ip-filter` 的域名才返回 Fake-IP。

`*.tailxxxxxx.ts.net` 已从 `fake-ip-filter` 中移除，因此：

- 公网代理域名仍可返回 `198.18.x.x`；
- Tailnet FQDN 由 macOS split DNS 直接返回真实 `100.x.x.x`；
- Tailnet 连接不需要经过 Mihomo 的 Fake-IP 映射。

## 正确的验证方式

### 1. 验证 Tailscale 没有接管全部 DNS

```bash
tailscale dns status
```

预期：

```text
Tailscale DNS: disabled
MagicDNS: enabled tailnet-wide
```

### 2. 验证 macOS split DNS

```bash
scutil --dns
```

预期看到：

```text
domain   : tailxxxxxx.ts.net
nameserver[0] : 100.100.100.100
```

### 3. 验证 MagicDNS 完整域名

macOS 上应使用系统解析接口：

```bash
dscacheutil -q host -a name device.tailxxxxxx.ts.net
```

当前实测结果：

```text
name: device.tailxxxxxx.ts.net
ip_address: 100.x.x.x
```

也可以直接对照 Quad100：

```bash
dig +short @100.100.100.100 device.tailxxxxxx.ts.net A
```

预期同样返回：

```text
100.x.x.x
```

### 4. 验证公网 DNS 进入 Mihomo

```bash
dscacheutil -q host -a name www.google.com
```

当前实测结果：

```text
name: www.google.com
ip_address: 198.18.x.x
```

`198.18.x.x` 说明普通公网 DNS 已被 Mihomo Fake-IP 接管，不是 DNS 污染地址。

也可以直接测试 Mihomo DNS listener：

```bash
dig +short @127.0.0.1 -p 1053 www.google.com A
```

预期返回 `198.18.x.x`。

### 5. 验证实际访问和规则命中

```bash
curl -I https://www.google.com
curl -I https://device.tailxxxxxx.ts.net:<端口>
```

Google 的 Mihomo 日志应类似：

```text
www.google.com:443 match GeoSite(google) using 谷歌服务[新加坡节点]
```

如果 Tailnet 主机没有 HTTPS 服务，可以改用实际开放的 SSH、HTTP 或其他端口。

## 不要用这些结果误判

### `dig @127.0.0.1 -p 1053 <host>.tailxxxxxx.ts.net` 超时

这条命令明确绕过 macOS split DNS，强制把 Tailnet 查询交给 Mihomo。当前设计并不要求这条路径成功。

正确测试是：

```bash
dscacheutil -q host -a name <host>.tailxxxxxx.ts.net
```

### 普通 `dig <host>.tailxxxxxx.ts.net` 与 `dscacheutil` 结果不同

在 macOS 上，`dig`、`host` 和 `nslookup` 不一定使用系统的多 resolver/Super DNS 选择逻辑。验证 `/etc/resolver` 和 MagicDNS 时，以 `dscacheutil` 以及真实应用访问结果为准。

### Tailnet 短名称无法解析

关闭 `accept-dns` 后，Tailscale 不再自动给 macOS 注入 `tailxxxxxx.ts.net` 搜索域，因此：

```text
device                         -> 默认无法解析
device.tailxxxxxx.ts.net       -> 正常解析
```

当前方案优先使用完整域名。如果确实需要短名称，可以单独给活动网络服务增加 search domain，但这不是 MagicDNS FQDN 正常工作的必要条件。

## 健康状态检查表

满足以下条件即可认为配置正常：

- `tailscale dns status` 显示 Tailscale DNS disabled、MagicDNS enabled。
- `/etc/resolver/tailxxxxxx.ts.net` 指向 `100.100.100.100`。
- `scutil --dns` 显示 `tailxxxxxx.ts.net` 的独立 resolver。
- `dscacheutil` 查询 Tailnet FQDN 返回真实 `100.x.x.x`。
- `dscacheutil` 查询 Google 等公网代理域名返回 `198.18.x.x`。
- `route -n get 100.100.100.100` 指向当前 Tailscale `utunN` 接口。
- Tailscale subnet 地址仍经 Tailscale 路由。
- Google 实际请求命中 `GeoSite(google)`，公网出口不是中国大陆。

典型健康状态：

```text
tailscale dns status
-> Tailscale DNS: disabled
-> MagicDNS: enabled

dscacheutil -q host -a name device.tailxxxxxx.ts.net
-> 100.x.x.x

dscacheutil -q host -a name www.google.com
-> 198.18.x.x

route -n get 100.100.100.100
-> interface: utunN
```

## 排障顺序

1. `tailscale status --json`
2. `tailscale dns status`
3. `cat /etc/resolver/tailxxxxxx.ts.net`
4. `scutil --dns`
5. `route -n get 100.100.100.100`
6. `dig +short @100.100.100.100 <host>.tailxxxxxx.ts.net A`
7. `dscacheutil -q host -a name <host>.tailxxxxxx.ts.net`
8. `dscacheutil -q host -a name www.google.com`
9. 检查 Clash Verge Rev 生成配置中的 `dns`、`tun` 和 `route-exclude-address`
10. 查看 Mihomo 实时日志确认 Google 的规则和节点

判断原则：

- Quad100 直查失败：先查 Tailscale/MagicDNS。
- Quad100 直查成功，但 `dscacheutil` 失败：先查 `/etc/resolver` 和 macOS DNS 缓存。
- Tailnet FQDN 正常但服务无法连接：检查 ACL、对端在线状态、防火墙和服务端口。
- 公网域名没有返回 Fake-IP：检查 Mihomo TUN 与 `dns-hijack`。
- Google 命中最终 `MATCH` 而不是 `GeoSite(google)`：检查系统查询是否绕过了 Mihomo。
