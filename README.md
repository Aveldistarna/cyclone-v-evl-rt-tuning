# Cyclone V SoC + EVL (Xenomai 4) — CPU1 Real-Time Isolation Patches

Intel Cyclone V SoC (DE10-Nano) 上で EVL (Xenomai 4) を動かして HILS (Hardware-In-the-Loop Simulation) システムを構築する際に、CPU1 上の OOB (Out-Of-Band) リアルタイムスレッドから **全ての周期的な Linux カーネル干渉を排除する**ためのカーネルパッチ集です。

0003〜0006 は、CPU0 で exec や mprotect が起きるたびに全 CPU に同報されるキャッシュ・TLB の保守操作を減らし、CPU1 のループの揺れを 0.1μs（80 サイクル）以下に抑えるためのものです。

## 背景

EVL は Linux カーネル上に co-kernel として動作するリアルタイム拡張です。`isolcpus=1 nohz_full=1 rcu_nocbs=1` を指定すれば CPU1 を OOB スレッド専用にできる…はずなのですが、実際には次のような周期的な Linux カーネル処理が CPU1 に IPI を送信し、OOB スレッドの実行時間を奪っていました：

- **proxy tick** が CPU1 で発火 → 27μs 級のスパイク
- **`sched_tick_remote()`** が 1Hz で `system_unbound_wq` 経由で実行 → CPU1 に reschedule IPI
- **`dl_task_timer()`** (SCHED_DEADLINE bandwidth replenishment) が hrtimer から発火 → CPU1 に function call IPI

これらは `nohz_full` でも `isolcpus` でも止められず、次のような **約 1 秒周期で 27〜35μs のスパイク** を引き起こしていました：

```
=== Spike Timing Analysis (compute only, 32M loops, 5.4 sec) ===
Spikes >10000 ns: 10
  #  loop_idx     offset(ms)   duration(ns)   gap(ms)
   0     5626833       949.96 ms       69703 ns      0.00
   1     5631732       950.84 ms       35165 ns      0.86
   2     5637449       951.84 ms       12717 ns      1.00
   ...
```

このパッチを適用すると、CPU1 はほぼ完全な「ベアメタル状態」になり、定期割り込み・IPI・スケジューラ tick が全て止まります。

```
=== Spike Timing Analysis (compute only, 32M loops, 5.4 sec) ===
Spikes >10000 ns: 0
```

## 動作環境

- **ボード**: Intel Cyclone V SoC (DE10-Nano)
- **CPU**: ARM Cortex-A9 dual-core @ 800 MHz
- **カーネル**: Linux 6.12.67 + EVL (Xenomai 4) co-kernel
  - ベース: [linux-evl](https://source.denx.de/Xenomai/xenomai4/linux-evl) commit `a0119f32d` (`evl/v6.12.y` ブランチ相当)
  - 0001〜0006 は、タグ `v6.12.67-evl2-rebase`（commit `53a3aa19a9f5`）に当ててビルドし、動作を確認しました
- **必要なカーネルコンフィグ**:
  - `CONFIG_HZ=1000`
  - `CONFIG_NO_HZ_FULL=y`
  - `CONFIG_RCU_NOCB_CPU=y`
  - `CONFIG_EVL=y`
- **必要なブートパラメータ**:
  - `isolcpus=1 nohz_full=1 rcu_nocbs=1`

## パッチの一覧

番号順に当てます。0003〜0006 は ARM（ARMv7 SMP）だけの変更です。

| ファイル | 内容 |
|---|---|
| [`0001-cpu1-rt-isolation.patch`](patches/0001-cpu1-rt-isolation.patch) | CPU1 の proxy tick と remote tick を止める |
| [`0002-sched-deadline-...`](patches/0002-sched-deadline-skip-dl_server_start-on-isolated-CPUs.patch) | 隔離 CPU では fair server（`dl_server`）を起動しない |
| [`0003-ARM-...`](patches/0003-ARM-invalidate-the-I-cache-only-when-an-executable-folio-changed.patch) | 実行ページの I キャッシュ全体の無効化（全 CPU に同報）を、ページの中身が変わった後の最初の 1 回だけにする |
| [`0004-ARM-flush_cache_range-...`](patches/0004-ARM-flush_cache_range-no-I-cache-invalidate-all-for-non-aliasing-D-caches.patch) | 非エイリアスの D キャッシュなら、mprotect / munmap で I キャッシュ全体を無効化しない |
| [`0005-ARM-keep-user-TLB-...`](patches/0005-ARM-keep-user-TLB-maintenance-local-when-the-mm-only-ran-on-this-CPU.patch) | 1 つの CPU でしか動いたことのないプロセスの TLB 保守を、その CPU の中だけで行う |
| [`0006-ARM-defer-...`](patches/0006-ARM-defer-the-I-cache-invalidation-for-isolated-CPUs-not-running-the-mm.patch) | 新しい実行ページの I キャッシュ無効化を、そのプロセスを動かしていない隔離 CPU には同報せず、その CPU が次にプロセスを切り替えるときに自分で行う |
| [`diagnostic/...`](patches/diagnostic/ARM-diagnostic-tracer-for-broadcast-maintenance.patch) | 診断用（任意）。同報された保守操作を `/proc/bcast_trace` に記録する。普段はオフで、製品には不要 |

## 0001 の内容

[`patches/0001-cpu1-rt-isolation.patch`](patches/0001-cpu1-rt-isolation.patch) は 2 ファイル・3 箇所の最小限の変更です。

### 1. `kernel/evl/clock.c` — proxy tick の発火抑制

```c
if (unlikely(timer == &rq->inband_timer)) {
-    rq->local_flags |= RQ_TPROXY;
+    if (!cpu_is_isolated(evl_rq_cpu(rq)))
+        rq->local_flags |= RQ_TPROXY;
    continue;
}
```

EVL の `do_clock_tick()` は inband 用タイマーが期限切れになった際に `RQ_TPROXY` フラグを立てます。このフラグが立つと、後続の `evl_core_tick()` から `evl_notify_proxy_tick()` が呼ばれ、inband stage に同期 IRQ が配送されます。

isolated CPU では inband スケジューラを動かさないので、このフラグを立てる必要がありません。これだけで CPU1 上の twd 発火が 12回/5秒 → 0回/5秒 になります。

### 2. `kernel/sched/core.c` — `sched_tick_remote()` の停止

```c
static void sched_tick_remote(struct work_struct *work)
{
    ...
+    /* Skip entirely for isolated CPUs — no tick, no requeue. */
+    if (cpu_is_isolated(cpu))
+        return;
    ...
}
```

`sched_tick_remote` は `nohz_full` CPU に対して **1Hz で hard-coded** で実行される delayed work です。housekeeping CPU 上で動作し、ターゲット CPU の rq lock を取って統計更新と reschedule を実施します。

isolated CPU では即 return することで、tick 処理も requeue も完全に止めます。これにより 1 秒周期の IPI 源が消えます。

### 3. `kernel/sched/core.c` — `sched_tick_start()` の早期 return

```c
static void sched_tick_start(int cpu)
{
    if (housekeeping_cpu(cpu, HK_TYPE_TICK))
        return;

+    if (cpu_is_isolated(cpu))
+        return;
    ...
}
```

タスク enqueue 時に呼ばれて新しい remote tick work を起動する関数。isolated CPU では起動自体をスキップします。

## 効果

| 条件 | 10μs+ スパイク (5秒間) | proxy tick (CPU1) | twd (CPU1) |
|------|----------------------|-------------------|------------|
| パッチなし | 10〜14 | +10〜12 | +12〜14 |
| パッチ適用 | **0** | **+0** | **+0** |

CPU1 上では：

- ✅ proxy tick 発火: ゼロ
- ✅ scheduler tick (twd): ゼロ
- ✅ Function call IPI: ゼロ
- ✅ kworker / softirq: ゼロ (元から)
- ⚠️ Reschedule IPI: `evl_switch_oob/inband()` 遷移時のみ (計測ループ中はゼロ)

## 0002 の内容

`kernel/sched/deadline.c` — 隔離 CPU では fair server を起動しない

```c
void dl_server_start(struct sched_dl_entity *dl_se)
{
    struct rq *rq = dl_se->rq;

+    /* Skip DL server on isolated CPUs to avoid 1Hz IPI noise. */
+    if (cpu_is_isolated(cpu_of(rq)))
+        return;
    ...
}
```

0001 だけのカーネルでは、CPU1 のループが開始から 0.95 秒と 1.95 秒に 13〜30μs 止まり、そのたびに CPU1 へ Reschedule IPI と Function call IPI が 1 回ずつ届いていました。0002 を足すと、10 秒の計測 3 回で 10μs を超える停止は 0 回、ループ中の CPU1 への Reschedule IPI と Function call IPI も 0 回になりました。CPU0 も止まりません。

## 0003〜0006 の内容

0001・0002 で CPU1 への割り込みと IPI が止まっても、CPU0 で exec・mprotect・munmap などが起きるたびに、カーネルは I キャッシュや TLB の保守操作を全 CPU に同報します（ARMv7 の ICIALLUIS など）。受け取った CPU1 は I キャッシュを消されてループのコードを取り直したり、TLB の保守で止まったりします（I キャッシュの同報 1 回で約 285 サイクル）。0003〜0006 は、CPU1 に届く同報を減らします。

計測した効果（CPU1 の PMU のカウンタ）:

| パッチ | 条件 | 前 | 後 |
|---|---|---|---|
| 0003 | ループ開始後 30 秒の I キャッシュの取り直し | 26,204 回 | 122〜124 回 |
| 0004 + 0005 | CPU0 で `stress-ng --exec` を 20 秒: I キャッシュの取り直し | 301,166 回 | 94 回 |
| 0004 + 0005 | CPU0 で `stress-ng --mmap` / `--fork`: D 側の TLB による停止 | 負荷なしの 30〜40 倍 | 負荷なしの 1.2〜1.4 倍 |
| 0006 | CPU0 で drop_caches の後に初めてのコマンド（date、ls、python3、gcc、perl など）を実行: I キャッシュの取り直し | 約 42,500 回 | 約 120 回 |
| 0006 | 同上: 約 285 サイクルの遅い回 | 約 500 回 | 0 回 |

健全性の確認: gcc でのコンパイルと実行、python3、apt、`stress-ng`（mmap / fork / exec）が通りました。python3 を CPU0 で起動して計算の途中で CPU1 に移す確認を 31 回行い、結果はすべて一致し、SIGILL・SIGSEGV は出ませんでした。

## 全部入れたときの結果

0001〜0006 を入れたカーネルで、CPU1 の EVL スレッド（SCHED_FIFO なし）が OOB で CCNT を連続して読み、1 周の長さを測りました。最頻値は 26 サイクル（800 MHz）です。ジッターは「最大 − 最頻値」です。計測では、OOB に入った後に 200μs 空回しし、ループが使うデータを全部 L1 に読み込んでから計測を始めました。

| 条件 | ジッター |
|---|---|
| 20 秒 × 6 回、CPU0 は空き | 40〜62 サイクル（50〜78ns） |
| 10 分、CPU0 は空き | 44 サイクル（55ns） |
| CPU0 で drop_caches と、コマンドの起動 × 2 回 | 59 / 51 サイクル |
| CPU0 から DDR に 1.6 GB/s で書き続ける × 2 回 | 41 / 40 サイクル |

計測開始直後の 1〜2 回は、111〜207 サイクルになることがありました。計測ツールの空回しが本番とは別のコード（同じ処理の別のコピー）だったので、本番のコードが温まっていなかったためと見ています。上の表はそれを除いた値です。

## CPU1 のループを書くときの注意

1. `mlockall(MCL_CURRENT | MCL_FUTURE)` と、FPGA ブリッジなどの `mmap` は、OOB に入る前に済ませる。
2. `evl_switch_oob()` の後、本番と同じ関数を数百 μs 空回ししてから本番に入る。めったに使わないデータや分岐も 1 回通しておく（L1 に無い行を初めて触ると 150〜390 サイクル待たされる）。空回しから本番まで OOB を出ない。
3. ループのデータは L1 D キャッシュ（32KB）に収める。CPU0 が DDR に大量に書いているときは、L2 にヒットする読み出しでも、1% の回で 80 サイクルを超えました。
4. EVL スレッドに SCHED_FIFO を設定しない（下の「壊れる前提」）。
5. 初期化の後は、CPU1 から in-band のシステムコール（ファイル I/O、sleep、printf など）を呼ばない（下の「壊れる前提」）。ログは、CPU0 のスレッドが読む共有メモリに書く。
6. 開発中は `evl_set_thread_mode(efd, EVL_T_WOSS | EVL_T_HMSIG, NULL)` を設定しておくと、in-band に落ちたときに SIGDEBUG（SIGXCPU）で分かる。

## デメリット・注意事項

### 動作するが意味を失うもの

これらは **CPU1 を OOB 専用にする運用** であれば全て問題ありません：

- CPU1 上の **CFS タスクのタイムスライス管理**が機能しない
- CPU1 上の **load average 計算**が止まる（監視ツール表示が不正確）
- CPU1 上の **kworker / ksoftirqd** が動かない（`isolcpus=1` で元々動かない）
- CPU1 上の CFS タスクが RT タスクに CPU を取られ続けたときの救済（**fair server**）が効かない（0002）

### 壊れる前提

- **`isolcpus=1` を外すと壊れる**: パッチは isolated CPU 前提の設計
- **CPU1 で inband タスクを走らせると壊れる**: tick がないのでスケジューリングが機能しない

CPU1 は完全に「OOB 専用カーネル」のように扱う必要があります。具体的に見えた症状:

- **CPU1 の in-band でブロック I/O をすると、システム全体が止まることがある。** `taskset -c 1 ls -R`（drop_caches の後）で、CPU1 のブロック層のワーカー（kblockd）が MMC の待ち（`usleep_range`）から戻らず、すべてのディスク I/O が止まりました（2 回）。CPU1 の in-band のタイマー待ちは、戻らないことがあります。
- **EVL スレッドに SCHED_FIFO を設定すると、OOB の間ずっと CPU0 が止まる。** 優先度に関係なく（1 でも 90 でも）、10 秒の OOB の間、CPU0 で 1ms ごとに起きるスレッドが 9.99 秒起きませんでした。FIFO を設定しなければ止まりません。

### 0003〜0006 の前提

- Cortex-A9（ARMv7 SMP、非エイリアスの D キャッシュ）でだけ確認しています。
- 自分で命令を書き込むプログラム（JIT など）は、これまでどおり cacheflush（`__builtin___clear_cache()`）を呼ぶ必要があります。
- 0006 が効くのは、隔離していない CPU が 1 つだけの構成です。他にも隔離していない CPU があれば、これまでどおり同報します。

## 適用方法

```bash
cd /path/to/linux-evl
for p in /path/to/patches/000*.patch; do patch -p1 < "$p"; done
# 診断用の記録を使うときだけ
patch -p1 < /path/to/patches/diagnostic/ARM-diagnostic-tracer-for-broadcast-maintenance.patch
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8 zImage modules
```

ビルドして動作を確認したのは、診断用も含めた全部入りの組み合わせです。診断用を除いた組み合わせは、当たることだけ確認しています。

## 計測の再現方法

CCNT (Cortex-A9 PMU cycle counter) をユーザーランドから読めるように `ccnt_enable.ko` をロードした上で、CPU1 に固定した OOB スレッドで純粋な計算ループを回し、`ccnt_read()` を 32M 回サンプリングして 10μs を超える区間をスパイクとして記録しています。

## 参考資料

- [Intel Cyclone V HPS Technical Reference Manual](https://www.intel.com/content/www/us/en/docs/programmable/683126/)
- [EVL / Xenomai 4](https://evlproject.org/)
- [Linux kernel `nohz_full` documentation](https://www.kernel.org/doc/html/latest/timers/no_hz.html)

## License

GPLv2 (Linux kernel に準ずる)
