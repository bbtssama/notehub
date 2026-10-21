# Windows 双出口分流：无 IPv6 的 WiFi + 只有 IPv6 的计量网卡

> 场景：笔记本连着不计费的 WiFi（**没有 IPv6**），同时插着一块按流量计费的 USB 上网卡（**有电信/运营商 IPv6**）。
> 目标：日常全部走 WiFi；IPv6 只在真正需要时静默走计量卡；WiFi 有 IPv6 时自动改走 WiFi。

这套策略在 Windows 上实测可用，核心是 **路由度量 + DNS 层 AAAA 过滤 + 状态同步** 三层配合。

---

## 一、结论先行

| 条件 | IPv4 出口 | IPv6 出口 | DNS 行为 |
|---|---|---|---|
| WiFi 有 IPv6 | WiFi | **WiFi** | 正常双栈，AAAA 放行 |
| WiFi 无 IPv6 | WiFi | 计量卡 | 双栈域名只返回 A；纯 IPv6 域名才返回 AAAA |

两种模式自动切换，切换依据是「WLAN 是否有 Preferred 状态的全局 IPv6 地址 + IPv6 默认路由」。

---

## 二、为什么系统默认行为不对

### 1. 度量值决定默认路由

同一目的地，Windows 取 `RouteMetric + InterfaceMetric` 最小者。默认情况下 WiFi 和上网卡谁的度量小就走谁，通常不是你想要的那个。用 `route print -4` 看 `Metric` 列，用 `Get-NetIPInterface` 看接口度量。

### 2. IPv6 会抢在 IPv4 前面

只要系统里有可用的全局 IPv6，`getaddrinfo` 按 RFC 6724 会把 AAAA 排在 A 前面，浏览器 Happy Eyeballs 也会先试 IPv6。结果：**所有双栈站点都会走计量卡的 IPv6**，哪怕 IPv4 明明可以从 WiFi 走。

实测：`GetHostAddresses("www.qq.com")` 返回顺序是 `240e:...`（IPv6）在前，IPv4 在后。

### 3. Windows 只在「接口断链」时切换

WiFi 连着但没网（Portal 认证、路由器掉线）不会自动切到上网卡。原生没有可达性探测。

---

## 三、实现

### Step 0：先摸清现状

```powershell
route print -4
route print -6
Get-NetAdapter | Select-Object Name,InterfaceDescription,ifIndex,Status
Get-NetIPInterface -AddressFamily IPv4 | Sort-Object InterfaceMetric
Get-NetIPInterface -AddressFamily IPv6 | Sort-Object InterfaceMetric
Get-NetRoute -DestinationPrefix '0.0.0.0/0'
Get-NetRoute -DestinationPrefix '::/0'
Find-NetRoute -RemoteIPAddress 8.8.8.8      # 直接告诉你去某个 IP 会走哪张卡
```

### Step 1：固定度量

```powershell
# IPv4：WiFi 优先，计量卡垫底
Set-NetIPInterface -InterfaceAlias "WLAN"     -AddressFamily IPv4 -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "以太网 2" -AddressFamily IPv4 -InterfaceMetric 9000

# IPv6：先按「WiFi 无 IPv6」处理
Set-NetIPInterface -InterfaceAlias "以太网 2" -AddressFamily IPv6 -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "WLAN"     -AddressFamily IPv6 -InterfaceMetric 9000

# 有线网卡保持自动（插网线时优先）
Set-NetIPInterface -InterfaceAlias "以太网" -AddressFamily IPv4 -AutomaticMetric Enabled
```

### Step 2：改 IPv6 前缀优先级

RFC 6724 的 prefix policy 默认让 IPv6 优先（`::/0` = 40 > `::ffff:0:0/96` = 35）。把 `::/0` 压到 35 以下：

```powershell
netsh interface ipv6 set prefixpolicy prefix=::/0 precedence=30 label=1
```

恢复默认：`... precedence=40 label=1`。

> 注意：这一层只影响「地址排序」，不能阻止应用直接连 IPv6 字面量。真正把 AAAA 掐掉要靠下一层。

### Step 3：DNS 层过滤 AAAA（关键）

原则：**域名有 A 记录时，AAAA 查询返回空；只有纯 IPv6 域名才放行 AAAA。**

这样双栈站点在应用看来「没有 IPv6」，只能走 IPv4/WiFi；而真正只有 AAAA 的站点仍然能解析、走计量卡——满足「需要 IPv6 时静默可用」。

用 [mosdns](https://github.com/IrineSistiana/mosdns) 的 `prefer_ipv4` 插件实现（v5.3+ 内置）：

```yaml
log:
  level: info
  file: "C:\\Tools\\mosdns\\mosdns.log"

plugins:
  - tag: cache
    type: cache
    args:
      size: 8192

  - tag: forward
    type: forward
    args:
      concurrent: 1
      upstreams:
        - addr: "udp://223.5.5.5"
        - addr: "udp://119.29.29.29"

  - tag: main
    type: sequence
    args:
      - exec: $cache
      - exec: prefer_ipv4      # 有 A 就拦 AAAA
      - exec: $forward

  - tag: udp_server
    type: udp_server
    args:
      entry: main
      listen: "127.0.0.1:53"

  - tag: tcp_server
    type: tcp_server
    args:
      entry: main
      listen: "127.0.0.1:53"

  # IPv6 DNS 也走本地
  - tag: udp_server_v6
    type: udp_server
    args:
      entry: main
      listen: "[::1]:53"

  - tag: tcp_server_v6
    type: tcp_server
    args:
      entry: main
      listen: "[::1]:53"
```

装成服务：`mosdns.exe service install -d <dir> -c <config>`，然后 `mosdns.exe service start`。

### Step 4：让系统真的用上它

```powershell
# 所有网卡 IPv4 DNS 指向本地代理，备用一个公共 DNS
Set-DnsClientServerAddress -InterfaceIndex 6  -ServerAddresses "127.0.0.1","223.5.5.5"
Set-DnsClientServerAddress -InterfaceIndex 51 -ServerAddresses "127.0.0.1","223.5.5.5"

# IPv6 DNS 指向 ::1 —— 这一步最容易漏，见「坑 1」
netsh interface ipv6 set dnsservers name="以太网 2" source=static address=::1
netsh interface ipv6 set dnsservers name="WLAN"     source=static address=::1

# 关掉多宿主并行解析，避免其他虚拟网卡的 DNS 抢答
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" `
  -Name DisableSmartNameResolution -Value 1 -Type DWord

ipconfig /flushdns
```

### Step 5：自动适配（WiFi 有 IPv6 时改走 WiFi）

一个每 60 秒跑一次的服务，做三件事：

1. 检测 `WLAN` 是否有 **Preferred、非 fe80 的全局 IPv6 地址**，且存在 `::/0` 默认路由；
2. 有 → 模式 `dual`：WLAN IPv6 度量 10、卡 9000、prefix 40、DNS 配置去掉 `prefer_ipv4`；
3. 没有 → 模式 `v4pref`：卡 IPv6 度量 10、WLAN 9000、prefix 30、DNS 配置加回 `prefer_ipv4`。

只在模式变化时改配置并重启 mosdns，避免频繁抖动。判断逻辑（PowerShell）：

```powershell
$wlan = Get-NetAdapter | Where-Object { $_.InterfaceDescription -like '*Wireless*' -and $_.Status -eq 'Up' } | Select-Object -First 1
$wlanV6 = $false
if ($wlan) {
    $addr = Get-NetIPAddress -InterfaceIndex $wlan.ifIndex -AddressFamily IPv6 |
            Where-Object { $_.AddressState -eq 'Preferred' -and $_.PrefixOrigin -ne 'WellKnown' -and $_.IPAddress -notlike 'fe80*' }
    $rt = Get-NetRoute -InterfaceIndex $wlan.ifIndex -AddressFamily IPv6 -DestinationPrefix '::/0'
    if ($addr -and $rt) { $wlanV6 = $true }
}
```

**运行方式很重要**：不要用任务计划器。用 [WinSW](https://github.com/winsw/winsw) 之类把循环包装成 Windows 服务（本机踩坑见「坑 2」）。服务里跑一个 `while ($true) { 检测 + 同步; Start-Sleep 60 }` 即可。

---

## 四、验证清单

```powershell
# 1) 双栈域名：AAAA 应为 0，A 应有值
(Resolve-DnsName www.qq.com -Type AAAA | ? {$_.Type -eq 'AAAA'}).Count   # 0
(Resolve-DnsName www.qq.com -Type A    | ? {$_.Type -eq 'A'}).Count      # >0

# 2) 纯 IPv6 域名：AAAA 应有值
(Resolve-DnsName ipv6.taobao.com -Type AAAA | ? {$_.Type -eq 'AAAA'}).Count  # >0

# 3) 实际出口
Find-NetRoute -RemoteIPAddress 8.8.8.8        # 应指向 WiFi
Find-NetRoute -RemoteIPAddress 2400:3200::1   # 应指向计量卡
curl.exe -s -o NUL -w "%{remote_ip}" https://www.qq.com   # 应为 IPv4

# 4) IPv6 通道本身可用（用字面量绕开 DNS）
curl.exe -6 -s -o NUL -w "%{http_code}" -H "Host: www.qq.com" http://[240e:97c:2f:201d::12d]
```

---

## 五、踩坑记录

### 坑 1：运营商 RA 下发的 IPv6 DNS 会绕过本地代理

`Get-DnsClientServerAddress` 里，上网卡的 IPv6 DNS 往往显示运营商地址（如 `240e:56:4000::218`）。Windows DNS 客户端会优先用它，导致 `nslookup` 默认请求根本没走 `127.0.0.1`，AAAA 过滤形同虚设。

- `netsh interface ipv6 set dnsservers ... address=none` **清不掉**动态源；
- 必须给一个静态值覆盖：`source=static address=::1`，并让 mosdns 监听 `[::1]:53`。

**自查方法**：`nslookup www.qq.com` 的 `Server:` 行必须是 `::1` 或 `127.0.0.1`。

### 坑 2：任务计划器 + 杀毒软件会挂死

在装有金山毒霸（铠甲防御）的机器上，用任务计划器以 SYSTEM 运行 PowerShell / Python / netsh，任务状态永远 Running、不产出任何结果；最简单的 `cmd /c echo > file` 反而正常。加了 `/rl HIGHEST` 后连最小脚本都挂。

**结论**：这类后台同步一律用 Windows 服务（WinSW / NSSM），不要用任务计划器。

### 坑 3：`nslookup -port=` 不存在

Windows 的 `nslookup` 不支持 `-port`，`Resolve-DnsName` 也没有 `-Port`。测非 53 端口要自己发原始 UDP 查询（注意 DNS 报文是大端序）。

### 坑 4：`::/0` 的 precedence 不要设反

想 IPv4 优先就压到 35 以下（如 30）；想恢复正常双栈就回到 40。设错会导致所有站点行为与你预期相反。

---

## 六、回滚

```powershell
# 停服务
mosdns.exe service stop; mosdns.exe service uninstall
<ServiceWrapper>.exe stop; <ServiceWrapper>.exe uninstall

# 度量恢复自动
foreach ($a in "WLAN","以太网 2") {
    Set-NetIPInterface -InterfaceAlias $a -AddressFamily IPv4 -AutomaticMetric Enabled
    Set-NetIPInterface -InterfaceAlias $a -AddressFamily IPv6 -AutomaticMetric Enabled
}

# 前缀策略恢复默认
netsh interface ipv6 set prefixpolicy prefix=::/0 precedence=40 label=1

# DNS 恢复（按实际网关）
Set-DnsClientServerAddress -InterfaceIndex 6  -ServerAddresses "<WiFi网关>"
Set-DnsClientServerAddress -InterfaceIndex 51 -ServerAddresses "<计量卡网关>"
netsh interface ipv6 set dnsservers name="WLAN" source=dhcp
netsh interface ipv6 set dnsservers name="以太网 2" source=dhcp

Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" -Name DisableSmartNameResolution -ErrorAction SilentlyContinue
ipconfig /flushdns
```

---

## 七、边界

1. Windows 只在接口断链时切换路由；WiFi 连着但没网不会自动切。需要的话加连通性探测。
2. 应用自建 DoH（Chrome 手动开安全 DNS）会绕过本地 DNS 分流。
3. 路由器/NAS 的本地主机名不再由路由器解析，需要单独加转发规则。
4. 不区分「计费 WiFi」：切到手机热点等按量计费的 WiFi 时仍按 WiFi 优先。
5. 纯 IPv6 站点在任何模式下都会消耗计量卡流量——除非 WiFi 侧也启用 IPv6（那是最彻底的方案）。

---

## License

MIT
