# OSTEP 第 4 章作业解答：process-run.py 进程调度模拟实验

本仓库为《操作系统导论（Operating Systems: Three Easy Pieces）》第 4 章（进程抽象）Homework 的作业解答。使用 OSTEP 官方模拟器 `process-run.py`（来源：`remzi-arpacidusseau/ostep-homework` 仓库 `cpu-intro` 目录）实际运行验证，所有答案均通过 `-c -p` 标志确认，与模拟器输出完全一致。

## 运行环境与默认参数

- 模拟器：`process-run.py`（Python 3 环境）
- 默认 I/O 时长：`-L 5`（5 个时间单位）
- 默认切换策略：`-S SWITCH_ON_IO`（进程发出 I/O 时切换）
- 默认 I/O 完成行为：`-I IO_RUN_LATER`（I/O 完成时继续运行当前进程）
- `-c`：直接打印正确追踪；`-p`：在追踪后打印统计（需与 `-c` 配合使用）

计时约定：一条 I/O 指令的完整过程为“发出（1 单位）＋ I/O 执行（5 单位，进程处于 BLOCKED 阻塞态）＋ 完成处理 io_done（1 单位）”；CPU 指令每条占用 1 个时间单位。

## 第 1 题：CPU 利用率

运行 `./process-run.py -l 5:100,5:100`。CPU 利用率（CPU 使用时间的百分比）应该是多少？

**答案：100%。** 两个进程各有 5 条指令，Y=100 表示所有指令均为 CPU 指令，任何进程都不会发出 I/O。CPU 从第 1 个时间单位到第 10 个时间单位全程忙碌，没有空闲。

验证输出：

```
Time        PID: 0        PID: 1           CPU           IOs
  1        RUN:cpu         READY             1
  2        RUN:cpu         READY             1
  3        RUN:cpu         READY             1
  4        RUN:cpu         READY             1
  5        RUN:cpu         READY             1
  6           DONE       RUN:cpu             1
  7           DONE       RUN:cpu             1
  8           DONE       RUN:cpu             1
  9           DONE       RUN:cpu             1
 10           DONE       RUN:cpu             1

Stats: Total Time 10
Stats: CPU Busy 10 (100.00%)
Stats: IO Busy  0 (0.00%)
```

## 第 2 题：完成两个进程所需时间

运行 `./process-run.py -l 4:100,1:0`。进程 0 有 4 条全部使用 CPU 的指令，进程 1 只发出一次 I/O 并等待它完成。完成这两个进程需要多长时间？

**答案：11 个时间单位。** 进程 0 先运行 4 条 CPU 指令（4 单位）；随后进程 1 发出 I/O（1 单位）、I/O 设备执行 5 个单位（期间进程阻塞、CPU 空闲）、I/O 完成后还需 1 个单位处理完成（io_done）。合计 4＋1＋5＋1＝11。

验证输出：

```
Time        PID: 0        PID: 1           CPU           IOs
  1        RUN:cpu         READY             1
  2        RUN:cpu         READY             1
  3        RUN:cpu         READY             1
  4        RUN:cpu         READY             1
  5           DONE        RUN:io             1
  6           DONE       BLOCKED                           1
  7           DONE       BLOCKED                           1
  8           DONE       BLOCKED                           1
  9           DONE       BLOCKED                           1
 10           DONE       BLOCKED                           1
 11*          DONE   RUN:io_done             1

Stats: Total Time 11
Stats: CPU Busy 6 (54.55%)
Stats: IO Busy  5 (45.45%)
```

## 第 3 题：交换进程顺序

运行 `./process-run.py -l 1:0,4:100`。发生了什么？交换顺序是否重要？为什么？

**答案：总时间变为 7 个时间单位，顺序重要。** 进程 0 先发出 I/O（1 单位）后阻塞；在默认的 SWITCH_ON_IO 策略下，系统立即切换到进程 1，进程 1 的 4 条 CPU 指令在 I/O 进行期间执行；I/O 完成后进程 0 用 1 单位处理完成。合计 1＋5＋1＝7。

顺序重要的原因：把发出 I/O 的进程放在前面，使 I/O 与另一进程的 CPU 工作重叠，CPU 与 I/O 设备同时忙碌，总时间从第 2 题的 11 缩短到 7；若 CPU 进程在前，I/O 期间没有其他进程可用，CPU 只能空转。若采用 SWITCH_ON_END 策略，两种顺序的总时间都是 11，顺序不影响总时间（但资源利用率都很差）。

验证输出：

```
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED       RUN:cpu             1             1
  3        BLOCKED       RUN:cpu             1             1
  4        BLOCKED       RUN:cpu             1             1
  5        BLOCKED       RUN:cpu             1             1
  6        BLOCKED          DONE                           1
  7*   RUN:io_done          DONE             1

Stats: Total Time 7
Stats: CPU Busy 6 (85.71%)
Stats: IO Busy  5 (71.43%)
```

## 第 4 题：SWITCH_ON_END 策略

运行 `./process-run.py -l 1:0,4:100 -c -S SWITCH_ON_END`。会发生什么？

**答案：总时间 11，CPU 利用率 54.55%。** 进程 0 发出 I/O 后，系统不切换到进程 1，CPU 空转等待 I/O 完成（5 单位），随后 io_done 占用 1 单位，最后进程 1 运行 4 条 CPU 指令。I/O 进行的 5 个时间单位内 CPU 完全闲置，资源浪费明显。

验证输出：

```
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED         READY                           1
  3        BLOCKED         READY                           1
  4        BLOCKED         READY                           1
  5        BLOCKED         READY                           1
  6        BLOCKED         READY                           1
  7*   RUN:io_done         READY             1
  8           DONE       RUN:cpu             1
  9           DONE       RUN:cpu             1
 10           DONE       RUN:cpu             1
 11           DONE       RUN:cpu             1

Stats: Total Time 11
Stats: CPU Busy 6 (54.55%)
Stats: IO Busy  5 (45.45%)
```

## 第 5 题：SWITCH_ON_IO 策略

运行 `./process-run.py -l 1:0,4:100 -c -S SWITCH_ON_IO`。现在会发生什么？

**答案：总时间 7，CPU 利用率 85.71%，I/O 利用率 71.43%。** 进程 0 发出 I/O 的瞬间系统切换到进程 1，进程 1 的 4 条 CPU 指令与 I/O 并行执行；I/O 完成后进程 0 用 1 单位处理完成。同样的两个进程，仅改变切换策略就节省了 4 个时间单位：I/O 与 CPU 工作重叠，资源得到更充分利用。

验证输出：

```
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED       RUN:cpu             1             1
  3        BLOCKED       RUN:cpu             1             1
  4        BLOCKED       RUN:cpu             1             1
  5        BLOCKED       RUN:cpu             1             1
  6        BLOCKED          DONE                           1
  7*   RUN:io_done          DONE             1

Stats: Total Time 7
Stats: CPU Busy 6 (85.71%)
Stats: IO Busy  5 (71.43%)
```

## 第 6 题：IO_RUN_LATER 下的资源利用

运行 `./process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p`。系统资源是否被有效利用？

**答案：没有被有效利用。** 进程 0 有 3 条 I/O 指令，进程 1、2、3 各有 5 条 CPU 指令。IO_RUN_LATER 规定 I/O 完成时继续运行当前进程，发出 I/O 的进程不一定马上运行：进程 0 的第 1 次 I/O 在第 6 个时间单位完成，但系统继续依次运行进程 2、3，进程 0 直到第 17 个时间单位才处理完成并发出第 2 次 I/O；其后进程 0 独自运行，最后两次 I/O 期间又没有其他进程可用，CPU 也随之空闲。

验证统计：

```
Stats: Total Time 31
Stats: CPU Busy 21 (67.74%)
Stats: IO Busy  15 (48.39%)
```

I/O 设备利用率仅 48.39%，I/O 完成后进程 0 等待长达 11 个时间单位才得以运行，设备大量空转；进程 0 最后两次 I/O 期间 CPU 又出现 10 个单位的空闲，总时间被拖长到 31。

## 第 7 题：IO_RUN_IMMEDIATE 策略

相同进程改用 `-I IO_RUN_IMMEDIATE`。行为有何不同？为什么运行一个刚刚完成 I/O 的进程会是一个好主意？

**答案：总时间 21，CPU 利用率 100%，I/O 利用率 71.43%。** 每次 I/O 一完成，进程 0 立即运行并马上发出下一次 I/O，I/O 设备保持忙碌；各进程的 CPU 指令穿插在 I/O 期间执行，CPU 全程忙碌。

验证统计：

```
Stats: Total Time 21
Stats: CPU Busy 21 (100.00%)
Stats: IO Busy  15 (71.43%)
```

这是好主意的原因：刚完成 I/O 的进程最可能紧接着再次发出 I/O，立即运行它可以让 I/O 设备保持忙碌，并使 I/O 与后续 CPU 工作持续重叠，从而同时提高 CPU 与 I/O 两种资源的利用率。

## 第 8 题：随机进程实验

运行 `-s 1 -l 3:50,3:50`、`-s 2 -l 3:50,3:50`、`-s 3 -l 3:50,3:50`。尝试预测追踪记录如何变化；使用 IO_RUN_IMMEDIATE 与 IO_RUN_LATER、SWITCH_ON_IO 与 SWITCH_ON_END 时会发生什么？

每个进程有 3 条指令，每条指令以 50% 概率是 CPU 指令或 I/O 指令（Y=50）。三个种子实际生成的指令序列：

- 种子 1：进程 0 为 CPU、I/O、I/O；进程 1 为 CPU、CPU、CPU
- 种子 2：进程 0 为 I/O、I/O、CPU；进程 1 为 CPU、I/O、I/O
- 种子 3：进程 0 为 CPU、I/O、CPU；进程 1 为 I/O、I/O、CPU

三种配置下的运行统计（数值依次为总时间、CPU 利用率、I/O 利用率）：

| 配置 | 种子 1 | 种子 2 | 种子 3 |
| --- | --- | --- | --- |
| SWITCH_ON_IO＋IO_RUN_LATER（默认） | 15 / 53.33% / 66.67% | 16 / 62.50% / 87.50% | 18 / 50.00% / 61.11% |
| SWITCH_ON_IO＋IO_RUN_IMMEDIATE | 15 / 53.33% / 66.67% | 16 / 62.50% / 87.50% | 17 / 52.94% / 64.71% |
| SWITCH_ON_END | 18 / 44.44% / 55.56% | 30 / 33.33% / 66.67% | 24 / 37.50% / 62.50% |

预测规律与观察结论：

1. **切换策略的影响**：SWITCH_ON_IO 在进程发出 I/O 时立即切换，让其他进程的 CPU 工作与 I/O 重叠，总时间明显更短（例如种子 2 从 SWITCH_ON_END 的 30 缩短到 16）；SWITCH_ON_END 在每次 I/O 期间让 CPU 空转，总时间最长、CPU 利用率最低（33.33% 到 44.44%）。
2. **I/O 完成行为的影响**：两种行为只在“I/O 完成时另有可运行进程”的情况下产生差别。种子 1、种子 2 中两种行为结果完全相同（I/O 完成时另一进程已结束或同样处于阻塞）；种子 3 中 IO_RUN_IMMEDIATE 使进程 0 的最后一条 CPU 指令与进程 1 的第 2 次 I/O 重叠，总时间从 18 缩短到 17。
3. **追踪记录本身取决于随机指令序列**：先按种子推演出每个进程每条指令是 CPU 还是 I/O，再依据切换策略与 I/O 完成行为逐单位推演运行、就绪、阻塞三种状态，即可得到与 `-c` 输出一致的追踪。

## 实验结论汇总

| 题号 | 配置 | 总时间 | CPU 利用率 | I/O 利用率 | 关键结论 |
| --- | --- | --- | --- | --- | --- |
| 1 | 5:100,5:100 | 10 | 100.00% | 0.00% | 纯 CPU 进程，CPU 全程忙碌 |
| 2 | 4:100,1:0 | 11 | 54.55% | 45.45% | I/O 期间 CPU 空转 |
| 3 | 1:0,4:100 | 7 | 85.71% | 71.43% | 顺序重要：I/O 与 CPU 工作重叠 |
| 4 | 1:0,4:100（SWITCH_ON_END） | 11 | 54.55% | 45.45% | I/O 期间不切换，CPU 空转 |
| 5 | 1:0,4:100（SWITCH_ON_IO） | 7 | 85.71% | 71.43% | 等待 I/O 时切换，重叠利用资源 |
| 6 | 3:0,5:100×3（IO_RUN_LATER） | 31 | 67.74% | 48.39% | I/O 完成后未及时运行，资源利用不充分 |
| 7 | 3:0,5:100×3（IO_RUN_IMMEDIATE） | 21 | 100.00% | 71.43% | 立即运行刚完成 I/O 的进程，利用率高 |
| 8 | -s 1/2/3，3:50,3:50 | 15～30 | 33.33%～62.50% | 55.56%～87.50% | 结果取决于指令序列与调度策略 |

核心结论：一是调度策略影响资源利用，进程发出 I/O 时及时切换到其他进程（SWITCH_ON_IO），可使 I/O 与 CPU 工作重叠，显著缩短完成时间；二是 I/O 完成行为同样关键，让刚完成 I/O 的进程立即运行（IO_RUN_IMMEDIATE），可尽快发出下一次 I/O，保持设备忙碌；三是进程执行顺序也会影响重叠程度，安排 I/O 进程先行可让 I/O 与 CPU 并行。
