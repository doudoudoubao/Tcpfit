# Tcpfit

按每台机器实测推导的 TCP 调优工具. 不套用固定参数, 实测 BDP 与限速器拐点.

本仓库是 [Kylin010/tcpfit](https://github.com/Kylin010/tcpfit)(MIT, 作者 kylin010) 的一个分支,
基于上游 **v0.5.3**. 调优的算法和思路全部来自上游, **本分支只做一件事: 别把机器跑挂**.

## 这个分支改了什么

上游在小内存 VPS 上会出这几类问题, 都是实机截图里出现过的:

| 症状 | 根因 | 本分支的处理 |
|---|---|---|
| 拐点测速把机器打满, SSH 卡死/断线 | HTB 只有一个班道, iperf3 把队列灌满(fq limit 40960 ≈ 4.9 秒排队延迟), SSH 的包排在几万个数据包后面 | SSH/DNS/ICMP 单开 prio 0 班道, 队列只有 128 个包 |
| 断线后机器永久停在测试限速上 | 只有 INT/TERM/HUP 的 trap, 挡不住 SIGKILL / OOM / `set -u` 崩溃 | 独立会话的看门狗进程 + systemd 定时器双兜底, 心跳一停自动恢复 |
| **正常跑完**也会被留在最后一档限速上 | `qdisc_restore` 恢复一次后就把状态清空, 结尾那次恢复变成空操作 —— 每次自动 sweep 都会踩 | 恢复函数改成可重复调用, 收尾另走 `qdisc_release` |
| 一次调优跑掉 22.8 GB 流量 | 线性逐档扫 + 端口轮换(11) × 重试(3) 相乘, 单个数据点最多 33 次 iperf3 | 二分定位拐点 + 流量/时间硬预算 + 重试收敛到最多 4 次 |
| `margin: unbound variable` 之类崩溃 | `set -u` 下某条分支没赋值 | 全局变量集中声明 + EXIT 兜底 trap(崩了也先把网卡恢复) |
| 加 swap 报错 `[x] swap 大小请填 1-20...` 然后整个脚本退出 | 把 y/n 的答案喂给了大小校验, 且校验失败直接 `die` | 只问大小不问 y/n, 非法输入原地重问, 建不成也只是少一段提示 |
| swap 建到一半把磁盘写满 | `fallocate` 失败回退 `dd`, 全程不查剩余空间 | 建之前查磁盘/文件系统/容器, 任何一步失败都把半截文件删干净 |
| 小内存机跑着跑着进程被杀 | qdisc 队列按包数写死 40960, 469 MB 的机器上光队列就 160 MB | fq 队列长度、并发流数按内存和核数缩放 |

细节和实测数据见 [`docs/CHANGES.md`](docs/CHANGES.md).

## 安装

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/doudoudoubao/Tcpfit/main/tcpfit.sh)
```

跑完直接出菜单, 选 1 全自动. 脚本会装到 `/usr/local/bin/tcpfit`, 以后敲 `tcpfit` 即可.

## 菜单

```
   1. 一键调优   Auto-tune (recommended)   ~5 min
   2. 基础调优   Base tuning only          ~1 min
   3. 拐点测试   Policer sweep (bisect)    ~4 min
   4. 加 swap    Add swap (low-memory box)
   ────────────────────────────────────────────
   5. 查看状态   Status
   6. 端口验证   Verify port capability    ~1 min
   7. 回滚改动   Rollback all changes
   8. 检查更新   Check for updates
   9. 解除限速   Clear test leftovers
```

一键调优只问四个问题: 带宽、测速对端、机器用途、流量上限（**回车 = 不限**）. 确认之后跑到底不再打断.

流量默认不限 —— 拐点是这个工具的核心结论, 给个保守上限会把扫描从中间砍断、
把拐点读低. 开跑前会打印预估流量, 结束时会报实际消耗, 所以"不限"不等于"不告诉你花了多少".
有流量配额的话在那一问填个数字即可.

带宽那一问支持四种输入:

| 输入 | 行为 |
|---|---|
| 数字 | 按该带宽推导缓冲区, 然后实测拐点 |
| 回车 | 现场实测带宽, 然后实测拐点 |
| `m` | 直接填限速值, 跳过拐点扫描 |
| `0` | 不做整形 |

## 子命令

```bash
tcpfit detect                                     # 机器画像
tcpfit probe    --peer <近处iperf3服务器>          # 探测可用带宽
tcpfit tune     --role proxy --bw 500             # 基础调优
tcpfit sweep    --peer <近处iperf3服务器> --nominal 500
tcpfit shape    --rate 510                        # 应用整形
tcpfit shape    --off                             # 移除整形, 保留基础调优
tcpfit harden   --swap 2G                         # 加 swap
tcpfit verify   --peer <近处iperf3服务器>          # 测速验证
tcpfit status                                     # 当前配置
tcpfit guard                                      # 查看有没有测试残留
tcpfit guard --off                                # 一键解除测试限速（被卡住时用它）
tcpfit rollback                                   # 回滚全部改动
tcpfit update                                     # 检查更新
```

`sweep` 的保护性选项:

| 选项 | 默认 | 说明 |
|---|---|---|
| `--budget-gb N` | **0（不限）** | 出向流量硬上限. 默认不限 —— 拐点测不准就没意义, 额度由你显式决定 |
| `--max-minutes N` | 60 | 墙钟上限, 防卡死用; 正常扫描 3-5 分钟 |
| `--precision N` | 按带宽自动取 5-25 | 二分停止精度(Mbit) |
| `--dur N` | 8 | 每档测多少秒 |
| `--parallel N` | 1 | 并发流数, 会被内存/核数自动压低 |

环境变量: `TCPFIT_BUDGET_GB` `TCPFIT_MAX_MINUTES` `TCPFIT_GUARD_TTL` `MAX_PORT_TRIES` `NETTUNE_VERBOSE`.

## 测速期间 SSH 为什么不会断

整形结构不再是一条大队列, 而是分了班道:

```
1:  htb default 10
└ 1:1   总闸        rate = ceil = 整形值          ← 聚合上限在这一层
  ├ 1:5  控制班道   prio 0   保底 5%   leaf pfifo limit 128
  │        SSH / DNS / ICMP 走这里, 排队延迟以毫秒计
  └ 1:10 数据班道   prio 7   其余带宽  leaf fq (逐包 pacing)
           iperf3 和其他一切走这里
```

`1:5` 的入口由 `tc u32` 过滤器指: TCP 22 端口(双向)、`sshd -T` 报的实际监听端口、
当前 SSH 会话的对端 IP、DNS 53、ICMP/ICMPv6. 开机时由 `tcpfit-qdisc.service`
重新探一遍 sshd 端口, 改过端口也不用重跑 tcpfit.

这条规则同时用于**测试期**和**最终应用**的整形 —— 生产环境里代理把出口打满时,
SSH 一样不会被自己的流量顶掉线.

## 万一还是被限速卡住了

```bash
tcpfit guard --off
```

它会收掉残留的 iperf3、删掉根 qdisc、把常驻整形(如果配过)重新装回去.
正常情况下用不到 —— 看门狗会在 90 秒内自己干完这件事.

## 它改了什么

| 类别 | 参数 |
|---|---|
| 拥塞控制 | `tcp_congestion_control=bbr` + `default_qdisc=fq` |
| 缓冲区 | `tcp_rmem` / `tcp_wmem` / `rmem_max` / `wmem_max` / `tcp_mem` |
| 窗口 | `tcp_window_scaling` / `tcp_moderate_rcvbuf` / `tcp_adv_win_scale` |
| 队列 | `netdev_max_backlog` / `netdev_budget` / `somaxconn` 等 |
| 连接 | `tcp_tw_reuse` / `tcp_fin_timeout` / `ip_local_port_range` 等 |
| 起步 | `tcp_slow_start_after_idle=0` / `initcwnd 32` |
| 出向整形 | HTB 全局上限 + 控制班道 + fq 叶子 pacing |

共 32 个 sysctl 参数. 缓冲区和整形值按每台机器实测推导, 不是固定值.

## 拐点扫描怎么工作

先不限速跑一次, 看有没有东西在打你:

| 结果 | 动作 |
|---|---|
| 丢包低 | 没有限速器, 不整形 |
| 丢包高 | 有限速器, 在 `[0.95×实测吞吐, 上界]` 区间二分找拐点 |
| 吞吐 > 2500 Mbit | 超出扫描上限, 不扫（可用 `--cap` 调整） |

拐点在"不限速吞吐"的**上面** —— 打穿限速器会让吞吐掉下来, 所以往上找.

二分的第一步是**先验下界**: 一来二分需要一个已知干净的下界, 二来要用它取本底丢包率.
(上游是线性扫描, 第一档天然在拐点之下, 顺手就当了本底; 二分的第一档在上界,
那是拐点之上, 拿它当本底会让后面每一档都判不出跳变 —— 这个坑本分支踩过并修好了.)

同样 5 Mbit 精度下, 500M 的机器从约 20 档降到约 8 次测量.

## 回滚

```bash
tcpfit rollback                # 按快照逐项写回, 不是恢复默认
tcpfit rollback --purge-swap   # 同时删掉 harden 建的 /swapfile
tcpfit shape --off             # 只去掉整形
```

首次改动前自动存快照到 `/var/lib/tcpfit/pre-tune.snapshot`, 记录全部 32 项参数的原始值.

改动只落在这些文件, 不碰 `/etc/sysctl.conf`:

```
/etc/sysctl.d/99-tcpfit.conf
/etc/systemd/system/tcpfit-qdisc.service
/usr/local/sbin/tcpfit-qdisc.sh
/etc/networkd-dispatcher/routable.d/50-tcpfit-initcwnd
/etc/modules-load.d/tcpfit-bbr.conf
/var/lib/tcpfit/
```

## 已知限制

- 瓶颈在国际链路而非端口时, 整形不会带来提升, 但输出看起来一切正常
- 需要 Linux + systemd + iproute2. OpenVZ/LXC 上 `tc` 和 `initcwnd` 可能受限
- 容器(LXC/OpenVZ/Docker)里不能自己开 swap, 脚本会明确告诉你而不是丢一个 EPERM
- `sweep` 需要一台近处的 iperf3 对端
- 流量预算是在每次测量【开始前】按该档预计用量做前瞻判断的, 所以是硬上限,
  但如果链路实际比设定速率跑得还快(不太可能), 仍可能小幅超出

## 多机（未上线）

多机编排继承自上游, 没有在真实环境验证过, 不建议使用.

## 许可证

[MIT](LICENSE) — 版权归上游作者 Kylin010, 本分支沿用同一许可证.
