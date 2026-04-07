# Cyclone V SoC + EVL (Xenomai 4) — CPU1 Real-Time Isolation Patches

Intel Cyclone V SoC (DE10-Nano) 上で EVL (Xenomai 4) を動かして HILS (Hardware-In-the-Loop Simulation) システムを構築する際に、CPU1 上の OOB (Out-Of-Band) リアルタイムスレッドから **全ての周期的な Linux カーネル干渉を排除する**ためのカーネルパッチ集です。

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
- **必要なカーネルコンフィグ**:
  - `CONFIG_HZ=1000`
  - `CONFIG_NO_HZ_FULL=y`
  - `CONFIG_RCU_NOCB_CPU=y`
  - `CONFIG_EVL=y`
- **必要なブートパラメータ**:
  - `isolcpus=1 nohz_full=1 rcu_nocbs=1`

## パッチの内容

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

## デメリット・注意事項

### 動作するが意味を失うもの

これらは **CPU1 を OOB 専用にする運用** であれば全て問題ありません：

- CPU1 上の **CFS タスクのタイムスライス管理**が機能しない
- CPU1 上の **load average 計算**が止まる（監視ツール表示が不正確）
- CPU1 上の **kworker / ksoftirqd** が動かない（`isolcpus=1` で元々動かない）

### 壊れる前提

- **`isolcpus=1` を外すと壊れる**: パッチは isolated CPU 前提の設計
- **CPU1 で inband タスクを走らせると壊れる**: tick がないのでスケジューリングが機能しない

CPU1 は完全に「OOB 専用カーネル」のように扱う必要があります。

## 適用方法

```bash
cd /path/to/linux-evl
patch -p1 < /path/to/0001-cpu1-rt-isolation.patch
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8 zImage modules
```

## 計測の再現方法

CCNT (Cortex-A9 PMU cycle counter) をユーザーランドから読めるように `ccnt_enable.ko` をロードした上で、CPU1 に固定した OOB スレッドで純粋な計算ループを回し、`ccnt_read()` を 32M 回サンプリングして 10μs を超える区間をスパイクとして記録しています。

## 参考資料

- [Intel Cyclone V HPS Technical Reference Manual](https://www.intel.com/content/www/us/en/docs/programmable/683126/)
- [EVL / Xenomai 4](https://evlproject.org/)
- [Linux kernel `nohz_full` documentation](https://www.kernel.org/doc/html/latest/timers/no_hz.html)

## License

GPLv2 (Linux kernel に準ずる)
