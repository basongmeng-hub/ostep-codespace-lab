# OSTEP 第 27 章作业解答：插叙·线程 API（helgrind 实验）

对应《操作系统导论》（Operating Systems: Three Easy Pieces）第 27 章 Homework (Code)：分析多线程程序，并用 **helgrind**（valgrind 调试套件中的线程错误检测工具）找出其中的并发问题，回答教材 9 道题目。

## 运行环境与工具

本解答在 Linux（WSL Ubuntu）上实际编译运行验证，全部输出取自真实运行：

- gcc 15.2.0（编译参数 `-Wall -pthread -g`）
- valgrind 3.26.0（helgrind 工具）
- make

构建全部程序：

```bash
make
```

helgrind 用法：

```bash
valgrind --tool=helgrind ./main-race
```

## 文件清单

| 文件 | 说明 |
| --- | --- |
| main-race.c | 简单数据竞争 |
| main-deadlock.c | 简单死锁（锁序反转） |
| main-deadlock-global.c | 全局锁解决死锁 |
| main-signal.c | 用标志变量 done 自旋同步 |
| main-signal-cv.c | 用条件变量同步 |
| common_threads.h | 带错误检查的 pthread 包装宏 |
| Makefile | 构建脚本 |
| main-race-oneline.c | 第 2 题变体：删去 main 中的一次 balance++ |
| main-race-lockone.c | 第 2 题变体：只在 worker 内加锁 |
| main-race-lockboth.c | 第 2 题变体：两处更新都加锁 |
| results/*.txt | 全部实验原始输出（证据，逐文件对应） |

## 第 1 题

**数据竞争在哪。** main-race.c 中 worker（第 8 行）与 main（第 15 行）都对全局变量 balance 执行 `balance++`，两处均无锁保护。`balance++` 不是原子操作（读-改-写三步），两个线程交错执行时最终值可能小于 2——这就是明显的数据竞争。

**helgrind 报告。** 运行 `valgrind --tool=helgrind ./main-race`：

```
==448== Possible data race during read of size 4 at 0x4004014 by thread #1
==448== Locks held: none
==448==    at 0x4001233: main (main-race.c:15)
==448== This conflicts with a previous write of size 4 by thread #2
==448== Locks held: none
==448==    at 0x40011BE: worker (main-race.c:8)
==448== Address 0x4004014 is 0 bytes inside data symbol "balance"
...
==448== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 1 from 1)
```

**是否指向正确的行？** 是。它精确报告 main-race.c:15（main 的读/写）与 main-race.c:8（worker 的写）之间的冲突。**还给出哪些信息？** 冲突双方的访问类型与字节大小（read/write of size 4）、冲突时各线程持有的锁（Locks held: none）、线程编号与完整调用栈、以及冲突地址对应的数据符号（balance）。

## 第 2 题

删除一处、只锁一处、两处都锁三种情况的实际结果：

1. **删除一行**（main-race-oneline.c，去掉 main 的 `balance++`）：只有一个线程访问 balance → `ERROR SUMMARY: 0 errors`，竞争消失。
2. **只锁一处**（main-race-lockone.c，仅 worker 内加锁）：竞争仍在——worker 的写在锁内（Locks held: 1），main 的读/写无锁（Locks held: none）→ `ERROR SUMMARY: 2 errors`。
3. **两处都锁**（main-race-lockboth.c）：两处更新都在同一把锁内 → `ERROR SUMMARY: 0 errors`，竞争消除。

结论：临界区两侧必须用同一把锁同时保护；只保护一侧等于没有保护。

## 第 3 题

**问题所在。** main-deadlock.c 的 worker 按参数选择加锁顺序：arg==0 时先锁 m1 再锁 m2（第 10–11 行），arg==1 时先锁 m2 再锁 m1（第 13–14 行）。若两个线程交错执行，可能形成：线程 0 持有 m1 等待 m2、线程 1 持有 m2 等待 m1——互相等待对方释放，谁也无法继续，即**死锁**（锁序反转 / 循环等待）。实际运行结果依赖调度时机，可能挂起也可能恰好完成（本机直接运行未挂起，输出为空）。

## 第 4 题

运行 `valgrind --tool=helgrind ./main-deadlock`：

```
==458== Thread #3: lock order "0x4004040 before 0x4004080" violated
==458== Observed (incorrect) order is: acquisition of lock at 0x4004080
==458==    at ... worker (main-deadlock.c:13)
==458==  followed by a later acquisition of lock at 0x4004040
==458==    at ... worker (main-deadlock.c:14)
==458== Required order was established by acquisition of lock at 0x4004040
==458==    at ... worker (main-deadlock.c:10)
==458==  followed by a later acquisition of lock at 0x4004080
==458==    at ... worker (main-deadlock.c:11)
==458== Lock at 0x4004040 is 0 bytes inside data symbol "m1"
==458== Lock at 0x4004080 is 0 bytes inside data symbol "m2"
...
==458== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 10 from 10)
```

helgrind 报告"**锁序违反**"（lock order violated）：两条路径以相反顺序获取 m1、m2，构成死锁可能；并指出 4 个相关行号（10、11、13、14）与两个锁的符号名（m1、m2）。

## 第 5 题

**代码是否还有同样问题？** 没有。main-deadlock-global.c 在最外层增加全局锁 g：worker 先锁 g，再锁 m1/m2。任意时刻只有一个线程能持有 g 进入内层，两个线程不可能同时分别持有 m1 与 m2，锁序循环永远不会发生——**代码是安全的**。

**helgrind 是否报同样错误？** 仍然报（`ERROR SUMMARY: 1 errors`，锁序 violated，指向第 12–13 行与第 15–16 行）。原因：helgrind 只根据观测到的加锁顺序推断"潜在"死锁，不理解全局锁使该循环不可能发生的语义。

**这说明了什么？** helgrind 这类工具是**保守**的，会报告"潜在问题"而非"必然错误"，因此可能产生**误报（false positive）**。它能帮你定位可疑的锁序，但最终正确性仍需人工判断。

## 第 6 题

**为什么不高效。** main-signal.c 的父线程用 `while (done == 0);` **自旋（忙等）**等待子线程。父线程在等待期间占满一个 CPU 核心空转；子线程工作得越久，父线程把这段时间全部浪费在自旋上，无法做任何有用工作。

## 第 7 题

运行 `valgrind --tool=helgrind ./main-signal`：

```
==471== Possible data race during read of size 4 at 0x4004014 by thread #1
==471==    at 0x4001242: main (main-signal.c:16)
==471== This conflicts with a previous write of size 4 by thread #2
==471==    at 0x40011C8: worker (main-signal.c:9)
==471== Address 0x4004014 is 0 bytes inside data symbol "done"
...
==471== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 104 from 32)
```

报告 1 个错误：子线程写 done（第 9 行）与父线程读 done（第 16 行）存在**数据竞争**。程序输出碰巧正确（先打印 first 再打印 last），但同步方式存在数据竞争，**从正确性角度并不安全**：`done` 的读写没有同步保护，行为未定义（且自旋浪费 CPU）。

## 第 8 题

**为什么更优？正确性、性能，还是两者？——两者。** main-signal-cv.c 用"条件变量 + 锁"替代标志自旋：

- **正确性**：`done` 的读写都在锁保护下（signal_done 持锁置位并发信号，signal_wait 持锁检查、未就绪则 `pthread_cond_wait` 睡眠并释放锁），无数据竞争；
- **性能**：父线程睡眠而非自旋，不占用 CPU；子线程完成时通过信号唤醒它。

自旋版既有数据竞争（done 无同步保护），又忙等浪费 CPU，所以两版在正确性和性能两方面都有差距。

## 第 9 题

运行 `valgrind --tool=helgrind ./main-signal-cv`：

```
==477== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 7 from 7)
```

helgrind **不报告任何错误**（0 errors）：`done` 的读写全部在锁保护下，条件变量信号同样持锁进行，同步正确。

## 实验结果汇总

| 程序 | helgrind 错误数 | 错误类型 | 说明 |
| --- | --- | --- | --- |
| main-race | 2 | 数据竞争 | 两处无锁 balance++ |
| main-race-oneline | 0 | — | 只剩一个线程访问 |
| main-race-lockone | 2 | 数据竞争 | 只锁一侧 |
| main-race-lockboth | 0 | — | 两侧都锁 |
| main-deadlock | 1 | 锁序违反 | 潜在死锁 |
| main-deadlock-global | 1 | 锁序违反（误报） | 全局锁已防死锁 |
| main-signal | 1 | 数据竞争 | done 无同步 + 自旋 |
| main-signal-cv | 0 | — | 条件变量正确同步 |

原始输出见 `results/` 目录（race-hg.txt、deadlock-hg.txt、deadlock-global-hg.txt、signal-hg.txt、signal-cv-hg.txt、oneline-hg.txt、lockone-hg.txt、lockboth-hg.txt 及对应程序运行输出）。

## 结论

本次作业验证了三条要点：

1. **数据竞争**：临界区两侧必须用同一把锁同时保护，只锁一侧等于没锁（main-race 系列 2→0 的错误数变化）。
2. **死锁与工具局限**：死锁源于锁序反转，可用全局锁避免；但 helgrind 会保守地误报潜在锁序问题（main-deadlock-global 仍报 1 错误），工具报告需要人工甄别。
3. **线程同步**：线程之间应使用条件变量（正确且高效），避免标志自旋——main-signal 有竞争且浪费 CPU，main-signal-cv 0 错误且父线程睡眠等待。
