# 相对上游 v0.5.3 的改动

上游: [Kylin010/tcpfit](https://github.com/Kylin010/tcpfit) (MIT).
调优算法、参数推导、拐点思路全部来自上游, 本分支只解决"跑的过程中把机器搞挂"这一类问题.

每一条都对应一个实机上出现过的现象.

---

## 1. SSH 在拐点测速期间断线

**现象**: 选 3 拐点测试, 跑到一半 SSH 卡死, 敲什么都没反应, 最后连接超时.

**根因**: 上游的测试整形结构是

```
1: htb default 10
└ 1:10  rate = ceil = 测试速率     leaf: fq limit 40960 flow_limit 8192
```

只有一条班道, 出向的**所有**流量都进这条队列, 包括 SSH.
iperf3 一开跑就把队列灌满, `limit 40960` 是包数 —— 按 1.5 KB 一个包算:

| 测试速率 | 队列满时的排队延迟 |
|---|---|
| 100 Mbit | 4.9 秒 |
| 500 Mbit | 0.98 秒 |
| 1 Gbit | 0.49 秒 |

SSH 的每一个按键都要排在这几万个数据包后面. 100-300 Mbit 档位上这就是必断.

**改法**: 分班道, SSH 单走一条 prio 0 的短队列.

```
1:  htb default 10
└ 1:1   总闸        rate = ceil = 整形值            ← 聚合上限在这一层
  ├ 1:5  控制班道   prio 0  保底 5%  leaf: pfifo limit 128
  └ 1:10 数据班道   prio 7  其余     leaf: fq (逐包 pacing)
```

`1:5` 的入口由 `tc u32` 过滤器指定, 不依赖 iptables/nftables:

- TCP 22 双向
- `sshd -T` 报的实际监听端口(改过端口的机器靠这个)
- `$SSH_CONNECTION` 里当前会话的服务端端口
- 当前 SSH 客户端 IP 的全部流量(端口改得再怪也认得出)
- DNS 53(TCP)、ICMP、ICMPv6

pfifo 128 个包的队列在 100 Mbit 下是 15 毫秒, prio 0 保证它先出队.

**同一套结构也用在最终应用的整形上** —— 生产环境里代理把出口打满时,
SSH 一样不会被自己的流量顶掉线. 开机时 `tcpfit-qdisc.service` 会重新探一遍
sshd 端口, 所以改端口之后不用重跑 tcpfit.

代码: `build_shaper()` / `add_ctrl_filters()` / `detect_ctrl_ports()` / `write_qdisc()`

---

## 2. 脚本被杀之后, 机器永久停在测试限速上

**现象**: 测速时 SSH 断了, 重连之后机器又慢又丢包, `tc qdisc show` 看到一条陌生的 htb.

**根因**: 上游只有 `trap ... INT TERM HUP`. 三种死法它接不住:

| 死法 | 信号 trap | EXIT trap |
|---|---|---|
| Ctrl-C / SIGTERM / SSH 挂断 | ✅ | ✅ |
| `set -u` unbound variable / 语法错误 | ❌ | ✅ |
| SIGKILL / OOM killer / 宿主机硬重启 | ❌ | ❌ |

第二类就是截图里 `line 1738: margin: unbound variable` 那次 —— bash 直接退出,
不走任何信号 trap.

**改法**: 两层.

1. **EXIT trap** (`cleanup_all`) 接住 `set -u` 崩溃和一切正常/异常返回.
   实测确认 bash 在 unbound variable 上会执行 EXIT trap.
   顺序是先 `qdisc_restore` 再 `guard_disarm`, 不能反.

2. **外部看门狗** 接住 SIGKILL/OOM. 改 qdisc 之前先武装:
   - 一个独立 session 的 `/bin/sh` 进程, 主脚本每次测量前后打心跳(`spin_wait`
     里每约 3 秒一次, 测量进行中也不断).
   - 心跳停 `GUARD_TTL`(默认 90) 秒 → 认定主脚本已死 → 收掉残留 iperf3、
     删测试 qdisc、把常驻整形装回去、清掉心跳文件.
   - 主脚本正常收尾时删掉心跳文件, 看门狗看到文件没了直接退出, 什么都不做.
   - 再加一个 `systemd-run --on-active` 的定时器做硬兜底 —— 连看门狗进程
     一起被 OOM 杀掉时还能救回来.

   坑: 看门狗会继承主脚本用 fd 9 持有的 `flock`, 必须 `9>&-` 关掉, 否则
   看门狗活多久就替主脚本把锁攥多久, 下次运行会撞上"另一个 tcpfit 正在运行".
   这是本地实测踩出来的.

3. 万一还是被卡住, 给用户一条不用看懂 tc 的命令: `tcpfit guard --off`.

代码: `guard_arm()` / `guard_beat()` / `guard_disarm()` / `cleanup_all()` / `cmd_guard()`

---

## 2b. sweep 正常跑完之后, 机器仍被留在最后一档测试限速上

**现象**: 单独跑菜单 3(拐点测试), 显示 `qdisc restored`、给出建议整形值,
但机器实际还挂在扫描的最后一档限速上. 如果那一档正好是打穿限速器的那一档,
表现就是"测完之后网络变得又慢又丢包", 而屏幕上写着一切正常.

**根因**: `qdisc_restore()` 每次恢复完都把 `QSAVE_IFACE` 清空,
于是**第一次恢复之后的所有恢复都变成空操作**.

而 sweep 的自动流程中途恰好会恢复一次 —— 不限速裸测跑完那里:

```
    tc qdisc del ...; tc qdisc add ... root fq      # 裸测
    for _ in 1 2 3; do ures=$(run_iperf ...); done
    qdisc_restore          # ← 这一次把 QSAVE_IFACE 清空了
    ... 逐档扫描, 每档 apply_test_shaper ...
  trap - INT TERM HUP
  restore_qdisc            # ← 走到 [ -n "$QSAVE_IFACE" ] || return 0 直接返回, 什么也没做
```

这条路径在**每一次自动模式的 sweep** 上都会走到, 不需要任何异常.

**改法**: `qdisc_restore()` 不再清 `QSAVE_IFACE`, 变成可重复调用;
另加一个 `qdisc_release()`(恢复 + 交还)只在整个测试流程真正结束时调.
所有中途恢复用 `qdisc_restore`, 所有收尾用 `qdisc_release`.

打桩验证(`qdisc_save` → 装限速 → 恢复一次 → 再装一档 → 收尾):

```
  upstream  收尾后网卡: class htb 1:10 ... rate 640Mbit ceil 640Mbit   ← 残留
  fork      收尾后网卡: <空, 已恢复干净>
```

完整 sweep 跑完之后同样如此: 本分支 `tc class show` 为空.

---

## 3. 一次调优跑掉 22.8 GB 流量

**现象**: 结果页显示"出向(上传) 22.80 GB". 很多 VPS 一个月才 500 GB - 1 TB.

**根因**: 三个乘数叠在一起.

1. **线性逐档扫描**. 档数 = 区间/步长, 随带宽线性增长.
   2 Gbps 的机器要扫约 40 档, 每档满速 12 秒 —— 上游自己的注释里写了"跑掉 137 GB".
2. **重试与端口轮换相乘**. `run_iperf` 会把 `PORT_POOL` 的 11 个端口挨个试,
   外层还包了 3 次重试 → **单个数据点最多 33 次 iperf3**.
   截图二里那二十几行 `10s × 4 流 → speedtest.lax12...` 就是这个.
3. **无条件复测**. 每次疑似跳变都补测 2 次满速档.

**改法**: 四处一起收.

| 改动 | 效果 |
|---|---|
| 线性扫 → **二分** | 测量次数 O(区间/步长) → O(log₂(区间/精度)) |
| 端口轮换 11 → **3** (`MAX_PORT_TRIES`), 重试 3 → **2** | 单数据点最多 33 次 → **4 次** |
| 复测只在"刚过线"时做(丢包 < 10× 阈值) | 大丢包一次定性, 不再白烧一档 |
| 每档 12s → **8s** | 直接砍三分之一 |

**外加一条硬预算**: 每次测量【开始前】按该档的预计用量做前瞻判断,
装不下就不开跑. 所以 `--budget-gb` 是真正的上限, 不是"超了才发现".
时间上同样有 `--max-minutes`.

预算用尽不是失败: 会带着"已测到的最高干净档"给结论, 并明确标注
`ABORTED=budget`, 免得被误读成"这台机器没有限速器".

模拟实测(真拐点 530 Mbit, 不限速 481 Mbit, 5 Mbit 精度):

```
  Rate/Mbit  Goodput/Mbps   Retrans    Loss%  Verdict
  none                481     18934   5.6999  loss -- policer present
  456                 437         6   0.0020  clean (baseline)
  656                 460     11118   3.4998  loss spike
  556                 490     11843   3.4997  loss spike
  506                 485         6   0.0018  clean
  531                 497     12013   3.5000  loss spike
  518                 497         6   0.0017  clean
  524                 503         6   0.0017  clean
  527                 505         6   0.0017  clean
  → 实测上限 527 Mbit（真值 530）, 共 9 次 iperf3
```

同样精度下线性扫需要约 20 档再加复测和细扫.

代码: `cmd_sweep()` 的二分段 / `budget_ok()` / `run_iperf()` / `confirm_spike()`

### 二分踩到的一个坑

`is_spike()` 有一条"必须明显高于本底(≥10×)"的规则, 用来避免把本身有损的线路
误判成拐点. 上游是线性扫描, 第一档天然在拐点【之下】, 顺手就当了本底.

二分的第一档在【上界】—— 那是拐点之上. 拿它当本底会让之后每一档都判不出跳变,
整个扫描全部读成 clean. 第一版就是这么写的, 模拟测试里当场暴露.

改成**先验下界**: 一来二分本来就需要一个已知干净的下界, 二来正好取本底.
下界要是也在丢包, 就把区间往下扩一倍再取一次.

---

## 4. `set -u` 下的 unbound variable

**现象**: `/dev/fd/63: line 1738: margin: unbound variable`, 跑完的结果一行都没打出来.

**根因**: `set -u` 下, 任何一条没走到的赋值路径都会让读取方直接退出.
上游在这上面栽过至少三次(GitHub #1 #2 的 knee/rate/margin, v0.4.3 的 swap).

**改法**:
- 所有跨函数读写的变量集中在一个块里给初值, 新增变量一律加到那里;
- EXIT trap 兜底 —— 再出这类崩溃, 至少网卡会被恢复, 并打印一行说明,
  而不是留下一台被限速的机器;
- 顺手修了 `shellcheck SC2183`: 有一处 `printf` 5 个占位符只喂了 4 个参数,
  输出会串列. 表格行统一走 `row()`;
- `usage()` 原本用 `sed -n '2,20p' "$0"` 读自己的注释头 —— `bash <(curl ...)` 跑时
  `$0` 是 `/dev/fd/63`, 内容已被 bash 读走, `-h` 打印一片空白. 改成 heredoc.

**注意**: 这里**故意没有**开 `set -e` 或 ERR trap. 脚本里大量命令的失败是预期内的
(`tc qdisc del` 没有 qdisc 时非 0、`pkill` 没进程时非 0、`modprobe` 没模块时非 0),
开了反而会在正常路径上乱退出.

---

## 5. 加 swap 报错然后整个脚本退出

**现象**:

```
创建 2G swap?  (Y/n) [y]: y
[*] Snapshot already exists, keeping the earliest one
[x] swap 大小请填 1-20 之间的整数（单位 GB）
root@localhost:~#
```

**根因**: 向导用 `confirm` 问了个 y/n, 然后把答案 `y` 当成大小喂给了 `cmd_harden --swap`,
校验不过 → `die` → **整个脚本退出**, 前面跑了十分钟的调优结论一行都没打出来.

上游 HEAD 已经把提问改成直接问大小, 但 `cmd_harden` 里仍有 4 条 `die` 路径,
在向导结尾/菜单里踩到任何一条都是同样的结果.

**改法**: 拆成 `harden_swap()`(核心, 只 return, 绝不 die) + `cmd_harden()`(命令行外壳).
向导和菜单都调前者. 顺便补齐上游完全没做的前置检查:

| 检查 | 不做会怎样 |
|---|---|
| **磁盘剩余空间**(要 size + 512 MB 余量) | `fallocate` 失败会回退 `dd`, `dd` 一路写到把根分区塞满, 留下一个半截 swapfile 和一台写不进东西的机器. 小盘 VPS 上必现 |
| **容器**(`systemd-detect-virt -c`) | LXC/OpenVZ/Docker 里 `swapon` 返回 `Operation not permitted`, 用户看着一句 EPERM 发懵 |
| **文件系统** | ZFS 上 swapfile 会死锁; overlayfs/aufs 不支持; tmpfs 上建 swap 是反效果; btrfs 必须 NOCOW + 不压缩且不能用 fallocate |
| **fallocate 结果核对** | 某些文件系统上 fallocate 造出稀疏文件, `mkswap` 能过但 `swapon` 报 `Invalid argument`. 现在核对实际大小, 对不上改用 dd |
| **失败清理** | 任何一步失败都 `rm -f` 掉半截文件, 磁盘不会被白占 |
| **fstab 顺序** | 上游先写 fstab 后 swapon; 失败时会留下一条指向不存在文件的记录, 下次开机 systemd 会卡在那儿等它. 现在 swapon 成功之后才写 |

输入解析也放宽了: `2` / `2G` / `2g` / `2GB` 都收, `2M` 和 `y` 明确拒绝
(`2M` 危险 —— fallocate 会建 2 MB 而回退的 dd 会建 2 GB, 差 1000 倍).

代码: `harden_swap()` / `swap_norm_gb()` / `disk_free_mb()` / `fs_type()` / `swap_max_safe_gb()`

---

## 6. 小内存机被队列和并发撑爆

**现象**: 469 MB 内存 + 无 swap 的机器, 测速跑一半代理进程被杀, 或 sshd 被 OOM killer 干掉.

**根因**: 三处按固定值写死, 完全没看机器多大.

| 上游写死 | 469 MB 机器上的实际代价 |
|---|---|
| `fq limit 40960` (包数) | 按 skb truesize ~4 KB 算, 光 qdisc 队列就能吃 **160 MB** |
| `iperf3 -P 4` | 单核机上 4 条流的 softirq + 用户态足够把 sshd 饿死 |
| `fq` 默认 `limit 10000`(不限速探测时) | 40 MB |

**改法**: 全部按内存和核数缩放.

```
fq limit = clamp(内存MB × 1024 × 3% / 4KB, 1000, 10240)
```

| 内存 | limit / flow_limit | 上游 |
|---|---|---|
| 469 MB | 3601 / 450 | 40960 / 8192 |
| 512 MB | 3932 / 491 | 40960 / 8192 |
| 1 GB | 7864 / 983 | 40960 / 8192 |
| ≥ 2 GB | 10240 / 1280 | 40960 / 8192 |

并发流数: 内存 ≤512 MB 或单核 → 1 条; ≤1 GB → 最多 2 条.
iperf3 加 `nice -n 10`, 让 sshd 在单核机上抢得过它.

**外加开测前的体检** (`sweep_preflight`): 内存 ≤768 MB 且无 swap 时,
先问要不要建 2G swap 再开测, 并说清楚不建的后果; 可用内存 < 128 MB 时
要用户明确确认. SSH 会话且不在 tmux/screen 里时, 说明两层保护分别是什么,
免得用户在卡顿的一瞬间去关终端.

代码: `fq_limits()` / `safe_streams()` / `sweep_preflight()` / `mem_available_mb()`

---

## 其它

- `shaper_rate()`: 分了班道之后 `tc class show | head -1` 不再可靠 —— 输出顺序不保证,
  抓到 `1:5` 会把 5% 的控制班道当成整形值报出来. 现在认准根类 `1:1`,
  找不到再回退到第一条 rate(兼容上游装的单班道结构).
- `MEASURES` 计数原本加在 `run_iperf` 里, 而它总是在 `$(...)` 子 shell 中调用,
  加了传不回来. 挪到 `measure_at`.
- `estimate_traffic_gb()` 按二分的固定次数重估(上游是按线性档数), 并据此给出
  默认预算 `default_budget_gb()`.
- 自动模式(带宽回车实测)下, 预算等实测出带宽之后再定 —— 否则 6 GB 的兜底值
  会把千兆机的扫描从中间砍断.

---

## 怎么验的

仓库里没有把模拟器一起提交(它只对开发有意义), 但结论如下, 都是本地实跑:

| 验证项 | 做法 | 结果 |
|---|---|---|
| 二分找拐点 | `tc`/`iperf3` 打桩模拟真拐点 530 Mbit | 收敛到 527 Mbit, 9 次 iperf3 |
| 没有限速器 | 裸测干净 | 1 次测量即停, `NO_KNEE=1` |
| 流量预算 | `--budget-gb 2` | 实际 1.74 GB, 带已测结论收工, 标 `ABORTED=budget` |
| 对端全程占线 | 打桩恒返回 busy | 6 次 iperf3 后放弃(上游同场景 33 次) |
| 看门狗 | 跑到装上测试限速时 `kill -9` 主脚本 | 心跳超时后自动 `tc qdisc del`, 恢复原状; 开火延迟实测 = TTL + 轮询间隔(≤3s) |
| sweep 收尾 | 正常跑完之后看 `tc class show` | 上游残留最后一档测试限速, 本分支为空 |
| EXIT 兜底 | 人为制造 unbound variable | 打印说明并恢复网卡, 不留限速 |
| swap 入参 | `2`/`2G`/`2g`/`2GB`/`y`/`0`/`99`/`2M`/空 | 前四个通过, 其余明确拒绝且不退出脚本 |
| 容器里加 swap | Docker 容器内 `harden --swap 2G` | 说明容器不能自己开 swap, 不产生半截文件 |
| shellcheck | `-S warning` | 只剩 4 条上游就有的 SC2034(未使用变量) |
