# OSTEP 第 7 章作业：CPU 调度（FIFO / SJF / RR）

本目录完成《操作系统导论》第 7 章 Homework (Simulation)：使用官方 `scheduler.py` 模拟器对比 **FIFO、SJF、RR** 三种调度策略在**平均响应时间、平均周转时间、平均等待时间**上的表现，并回答教材末尾 7 道作业题。

## 目录结构

```
hw/chapter7/
├── README.md       本文件：作业概述与结果汇总
├── analysis.md     7 道题的完整书面解答（含推导与模拟验证）
└── data/           每道题的模拟器原始输出（-c 计算结果）
    ├── q1_fifo_sjf.txt        Q1：三个 200 长度作业，FIFO vs SJF
    ├── q2_fifo_sjf.txt        Q2：100,200,300 作业，FIFO vs SJF
    ├── q3_rr.txt              Q3：100,200,300 作业，RR 时间片 1
    ├── q4_sjf_vs_fifo.txt     Q4：SJF 与 FIFO 周转相同的工作负载演示
    ├── q5_sjf_vs_rr.txt       Q5：SJF 与 RR 响应相同的工作负载/时间片演示
    ├── q6_sjf_trend.txt       Q6：SJF 响应时间随作业长度变化的趋势
    └── q7_rr_quantum.txt      Q7：RR 响应时间随时间片变化的趋势
```

## 结果汇总

**Q1（3×200）**：FIFO 与 SJF 完全一致 —— 平均响应 **200.00**，平均周转 **400.00**（等长作业下 SJF 退化为 FIFO 顺序）。

**Q2（100,200,300）**：FIFO 与 SJF 完全一致 —— 平均响应 **133.33**，平均周转 **333.33**（列表已升序，SJF 排序不变序）。

**Q3（100,200,300，RR q=1）**：平均响应骤降至 **1.00**，平均周转恶化至 **465.67** —— 性能与公平不可兼得。

**Q4**：作业长度列表已按升序排列（含全部等长）时，SJF 与 FIFO 周转时间相同；乱序负载（如 300,100,200）下 SJF 明显更优（333.33 vs 433.33）。

**Q5**：所有作业等长 L 且时间片 q = L 时，SJF 与 RR 响应时间逐作业相同；其余情况均不同。

**Q6**：SJF 平均响应时间随作业长度 **线性增长**（L=10→10.00，L=100→100.00，L=1000→1000.00）。

**Q7**：RR 平均响应时间随时间片 **线性增长**（q=1→1.00，q=5→5.00，q=10→10.00）；**N 个作业最坏情况响应时间 = (N−1)·q**。

## 关键命令

```bash
python3 scheduler.py -p FIFO -l 200,200,200 -c      # Q1
python3 scheduler.py -p SJF  -l 100,200,300 -c      # Q2
python3 scheduler.py -p RR   -q 1 -l 100,200,300 -c # Q3
python3 scheduler.py -p RR   -q 10 -l 10,10,10 -c   # Q5/Q7
```

- `-l` 指定作业长度列表，`-p` 选择策略（FIFO/SJF/RR），`-q` 设置 RR 时间片，`-c` 计算并显示全部指标，`-s` 指定随机种子（可复现）。
- 官方模拟器：`remzi-arpacidusseau/ostep-homework` → `cpu-sched/scheduler.py`。

## 数据可信性

`data/` 内所有文件均为在 Linux（WSL Ubuntu，python3）上真实运行官方 `scheduler.py` 的原始输出，未做任何修改；`analysis.md` 中的每个数字均可在对应数据文件中逐一核对。
