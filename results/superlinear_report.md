# 超线性加速比机制分析报告（P2 / N=5）

日期：2026-09-24 · 作者：超线性机制分析子任务
数据：全部数字来自官方评估器 `evaluate_scene_b` / `evaluate_scene_a`（`C:/shumo_live/a_data/code/`，一字未改）或其 bit-exact 复刻 FastEvalP2（改进方案均经官方二次复核）。反事实"hugcap" = 同一单核方案、容量 ×1000 跑官方 scene A（消除 spill 与内存复用链，VIRGIN credits → 0 内存依赖），只改 `capacity` 入参。

## 结论速览

1. **Q1（超线性来自单核 spill 爆炸吗？）——半对。** 超线性组 10 例拆成两类：
   - **spill 分子膨胀型（5/10：092/073/097/028/083）**：单核 spill 10~277MB，`sp_hugcap`（消掉容量压力后的速度比）全部跌到 **4.37–5.07**——超线性完全由分子（单核基准被 spill+内存链拉长）贡献，例：092 sp 5.97→hugcap 4.98，083 5.27→4.37。
   - **管线串行解锁型（4/10：084/023/089/021+059 混合）**：单核 spill≈0，hugcap 前后 sp 不变（084: 6.62→6.62）。机制：单核把 M 管与 V 管的活完全串行执行（sc_mk ≈ W_M+W_V），而多核**每核 M/V 双管线并发**（084 核2：busy M=312K + V=317K 挤进 378K span），吞吐突破 5 op-cycle/cycle。
2. **Q2（中段 4.6–4.9 卡住的主因）**——不是负载不均（中段组 imbalance 中位数 1.03，比超线性组还好），主因按占比：
   - **V 功饥饿 + 单核已部分重叠**：中段组 `W_V/W_M` 中位数 0.15、serial_ratio 1.06（单核已把 V 藏进 M 的空隙 6%），管线天花板 `5×(1+V/M)/serial` 中位数 **5.45**，其中 6 例天花板 <5.05（037: 4.99）——**结构上不可能超 5**，当前 4.88 已贴顶。
   - **有空间但被关键路径/跨核延迟/DDR 锁死**：中段组 wait_over_mk 中位 1.25（062: 119×），n_transfers 中位 14；天花板与现实的差距（中位 0.72）主要耗在逐边 500c 延迟、MTE 拷贝与每管全序停顿上——与既有"E 类编织 DAG 定律"一致。
   - 少数分子也被 spill 税（072/078/091 spill 12–109MB），但 hugcap 后仍 ≤4.84，不够越线。
3. **Q3（结构差异）**：超线性组的签名 = **浅层宽图（D_levels 8–31 vs 中段 26–36 vs 对照 22–124）+ 高并行度（par_cap 中位 144 vs 对照 23）+ 单核完全串行（serial≈1.00）+ 或 spill 大（serial<0.94 且 sc_spill>10MB）**。对照组卡在 V 主导深图（W_V/W_M 1.8–3.7）与低并行度（CP 界 binding，天花板 7.0 但只跑 3.8）。

## 最大发现（可操作）

**子图标签本身就是调度器**：评测器 `_prioritize_task_seq` 按"子图优先级"对 step1 的 DFS 序做稳定分桶。把每核的子图重标为 **batch(独立链组)主序 × level 次序** 的网格后，分桶序变成层主序，M/V 管线投影跨链交错 → 核内双管线并发被解锁，且 batch 宽度控制活跃张量宽度（避免 step2 spill）。**op→核 分配完全不动**：

| 案例 | 池 mk | 重标后 mk（B=48） | Δmk | sp 变化 | 官方复核 |
|---|---|---|---|---|---|
| case_008 | 95353（POOL_N5）/100603（refined） | **84420 / 80768** | **−11.5% / −19.7%** | 5.11→**5.78** / 4.85→**6.04** | bit-exact ✓ |
| case_095 | 420852 | **332341** | **−21.0%** | 4.95→**6.27** | bit-exact ✓（spill 0B） |

适用判据（全库扫描）：`pipe_ceiling>5.5` 且 `pool_sp<5.2` 且 **图可分解为 ≥20 条 ≤12 op 的独立链（弱连通分量）**且当前方案核内无 M/V 重叠（overlap_ratio≈1.0）→ 全库仅 **008、095** 两例严格满足（已全部验证，均值 +0.66/+1.32）。对链更粗（037: 39-op 链）、辫状单连通（063）、已被 refine2 调优过的（084: 重标反而 +47%）无效——如实报告为阴性。B 宽度有悬崖（095: B=48 零 spill，B=64 spill 22MB 崩溃），需 FastEval 扫 B∈[8,96]。

## 数据与方法

- 池值：`audit_20260924_latest/final_preview_q2.json`（分组口径）；池方案定位：`n5_push/gpu_search/coverage_report.md` 的 case→source 表 + FastEvalP2 复现 mk 校验；10 例（028/073/083/092/097/014/053/085/061/090）方案文件不在盘上（coverage 标注 archive/posthoc 源已失效），其 mc_mk 用 coverage 报告的官方 mk（标注 `coverage_mk_only`），单核侧证据（hugcap/spill）全部自测。
- 已知口径分歧：008 final_preview=4.85 但 POOL_N5 真身（`a_lab/runs/SMOKE-BAL5-ON/.../plan.json`）=5.11；091 同类（4.61 vs 4.94）。
- 组样本：超线性 10（>5.25）、中段 11（4.6–4.9）、对照 5（3.5–4.2），逐例 26 行见 `phase1_metrics*.csv`。
- 关键派生量：`serial_ratio=(W_M+W_V)/sc_mk`（≈1 表示单核把计算全串行）；`pipe_ceiling = sc_mk/(max(W_M,W_V)/5) = 5(1+V/M)/serial`（每核每管线 1 slot 的必要下界，非充分）；`overlap_ratio=(busy_M_max+busy_V_max)/span_mean`。

## Phase 1 组间对照（中位数；完整表见 `phase2_group_summary.txt`）

| 指标 | 超线性(10) | 中段(11) | 对照(5) |
|---|---|---|---|
| pool_sp | 5.45 | 4.77 | 3.80 |
| sc serial_ratio | **1.004** | 1.061 | 1.058 |
| sc_spill_MB | 7.5（5 例 10–277MB，4 例 0） | 0.0 | 2.2 |
| sp_hugcap | **4.97** | 4.72 | 3.66 |
| sp_if_nospill_est（只扣 spill 带宽） | 5.24 | 4.64 | 3.62 |
| W_V/W_M | 0.13（084 除外 0.98） | 0.15 | 1.8 |
| imbalance（local mk max/mean） | 1.02 | 1.03 | 1.23 |
| D_levels | 8–31（浅） | 26–36 | 22–124 |
| par_cap | 144 | 208 | 23 |

注意：hugcap 消除的是 spill **和**内存复用 WAR 链的总和，故 `sp_hugcap < sp_if_nospill_est`（只减 spill 传输时间）时说明容量还通过依赖链施压（例 059: 5.50→hugcap 4.89 但 nospill_est 5.31；092: 5.97→4.98/4.61）。

## Phase 2 三问证据

### Q1 分解表（超线性组逐例）

| case | pool_sp | sc_spill_MB | sp_hugcap | 判定 |
|---|---|---|---|---|
| 084 | 6.62 | 0 | 6.62 | 管线型（纯） |
| 023 | 5.28 | 0 | 5.28 | 管线型（纯） |
| 089 | 5.63 | 0 | 5.63 | 管线型（纯，ceiling 5.88 已贴顶） |
| 021 | 5.25 | 4.5 | 4.95 | 混合（容量贡献 ~0.3） |
| 059 | 5.50 | 1.7 | 4.89 | 混合（容量贡献 ~0.6） |
| 028 | 5.44 | 277.4 | 5.07 | spill 型 |
| 092 | 5.97 | 66.0 | 4.98 | spill 型 |
| 073 | 5.44 | 141.6 | 4.62 | spill 型 |
| 097 | 5.47 | 10.6 | 4.90 | spill 型 |
| 083 | 5.27 | 13.9 | 4.37 | spill 型 |

单核侧机制证据：084/089 的 sc_mk 与 W_M+W_V 之差 <0.1%（完全串行），且 hugcap（0 内存依赖、0 spill）下 mk 一点不变 → 串行来自 DAG 交替层结构 + 每管线全序，与容量无关。多核侧：084 核内 busy(M)+busy(V)/span = 1.75、008（改进前）= 1.00 —— 同一评测器内，重叠与否完全由子图结构决定。

### Q2 中段组逐层归因

- 贴顶不可救（ceiling<5.05）：037（4.88/4.99）、093（4.67/5.02）——V 功占比太小，5 条 M 管线就是全部产能。
- 有空间、被通信锁：063（ceiling 5.45，跑 4.60；wait_over_mk 2.5）、062（5.2 档，wait 119×mk）、078/072（跨核边 111–192 条）。
- 分子被 spill 税但不够：072（hugcap 4.59）、078（4.30）、091（4.84）、014（4.67）、076（4.68）。
- 破局示范：008（ceiling 9.75，跑 4.85）——不是通信问题，是**核内子图序**问题，重标后 6.04（见 Phase 3）。
- 中段组没有一例因负载不均损失 >8%（imbalance≤1.08）。

### Q3 结构签名

`ceiling_all100.csv`（全 100 例）：58 例 ceiling>5.5 但 pool<5.2（pipe 下界不 binding 的占多数，CP/延迟才是）；6 例 ceiling<5.05。超线性 = ceiling 中位 6.03 且**已贴顶**（headroom 0.38）；中段 ceiling 5.45、headroom 0.72；对照 ceiling 7.0 但 headroom 2.96（pipe 界远非 binding，par_cap 23 的深图 CP 锁死）。

## Phase 3 机制与 A/B 结果

**机制 A（已验证）：batch×level 子图重标**（`phase3_mechA2.py`，改 `node_to_subgraph`/`core_schedules`，op→核不动）
- 正例：008（−11.5% 真池 / −19.7% vs refined）、095（−21.0%），官方 bit-exact。
- 阴性（如实）：037（+1.2%）、093（0.0%）、063（+30.7%）、070（+37.6%）、076（+3.2%）、072/091（+0.9%）、022（+59%）、062（+129%）、078（+313%）、084（+47%，已被 refine2 隐式利用该机制）。原因：辫状单连通/链太粗/近天花板/已优化。
- 工程接入：对候选例 FastEval 扫 B∈{8..96} 取最优，keep-if-better，零风险。

**机制 B（设计，未实现）**：per-pipe 负载再平衡——对 063/062 类（空间在 M 管线均衡与跨核边上），把 refine2 的搬移目标从"总量均衡"改为 `min max_c(busy_M_c, busy_V_c)` 静态下界 + FastEval 判收，并优先消除关键路上的跨核边（合并相邻子图减边数）。本轮贪心搬移实现（`phase3_exploit.py`）未在预算内跑出正收益，留给下一轮。

## 文件清单

- `phase1_metrics.csv` / `_extra.csv` / `_extra2.csv`：26 例逐例全量指标
- `phase2_group_summary.txt` / `phase2_analyze.py`：组间对照与逐例表
- `ceiling_all100.csv`：全库管线天花板与分组
- `phase3_mechA2.py` / `phase3_mechA2.csv` / `phase3_mechA.py` / `phase3_exploit.py`：机制 A 引擎与 A/B 结果
- `phase1_collect.py` / `phase1_fix.py` / `phase1_hugcap_only.py`：采集管线
- `phase1_run.log`：方案定位 WARN 记录
