# ostep-codespace-lab

The course provides GitHub Codespaces for students to complete their homework and lab assignments, using *Operating Systems: Three Easy Pieces* as the primary textbook.
本课程为学生提供 GitHub Codespaces 以完成课后作业及实验任务，并以《Operating Systems: Three Easy Pieces》作为主要教材。

本仓库收录各章课后作业（Homework）的完成解答：书面分析、原始模拟/运行输出与可复现命令。

## 仓库结构（Repository Structure）

```
ostep-codespace-lab/
├── README.md                本文件：仓库总览与运行指南
└── hw/
    ├── process-run/         Ch.4  进程与 CPU 调度模拟（process-run.py，8 题）
    ├── chapter7/            Ch.7  CPU 调度（FIFO/SJF/RR，scheduler.py，7 题）
    └── setup/
        └── threads-api/     Ch.27 线程 API（helgrind 检查，9 题）
```

## 已完成作业（Completed Homework）

| 章节 | 主题 | 目录 | 主要工具 |
| --- | --- | --- | --- |
| Ch.4 | 进程运行与调度 | `hw/process-run/` | `process-run.py` |
| Ch.7 | CPU 调度：FIFO / SJF / RR | `hw/chapter7/` | `scheduler.py` |
| Ch.27 | 线程 API 与数据竞争 | `hw/setup/threads-api/` | C + `valgrind --tool=helgrind` |

## 运行代码（Run Commands）

### Ch.4 进程运行（hw/process-run）

```bash
# 预测：不运行 -c，先手算
python3 process-run.py -l 5:100,5:100
python3 process-run.py -l 4:100,1:0
python3 process-run.py -l 1:0,4:100 -c -S SWITCH_ON_IO

# 验证：加 -c 显示全部指标
python3 process-run.py -l 5:100,5:100 -c
```

### Ch.7 CPU 调度（hw/chapter7）

```bash
# Q1：三个长度为 200 的作业
python3 scheduler.py -p FIFO -l 200,200,200 -c
python3 scheduler.py -p SJF  -l 200,200,200 -c

# Q2：长度为 100,200,300 的作业
python3 scheduler.py -p FIFO -l 100,200,300 -c
python3 scheduler.py -p SJF  -l 100,200,300 -c

# Q3：RR 时间片 1
python3 scheduler.py -p RR -q 1 -l 100,200,300 -c

# 常用参数：-l 作业长度列表，-p 策略，-q 时间片，-c 显示结果，-s 随机种子
```

### Ch.27 线程 API（hw/setup/threads-api）

```bash
# 编译
make

# 用 helgrind 检测数据竞争 / 死锁
valgrind --tool=helgrind ./main-race
valgrind --tool=helgrind ./main-deadlock
valgrind --tool=helgrind ./main-signal

# 其余程序（oneline / lockone / lockboth / signal-cv 等）见 hw/setup/threads-api/
```

## 复现与验证

- 每个 `hw/*` 目录内含 `README.md`（结果概述）与 `analysis.md`（书面解答），原始输出保存在 `data/` 或 `results/` 中，所有数字均可逐项核对。
- 模拟器均为官方源码逐字节副本：`remzi-arpacidusseau/ostep-homework` 的 `cpu-sched/scheduler.py` 与 `process-run.py`。
- 教材章节与作业题：<https://pages.cs.wisc.edu/~remzi/OSTEP/>
