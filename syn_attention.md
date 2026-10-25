请将本轮 Attention 评估收敛为“一组最小完整测量”，实际执行并交付结果。先完成首组数据，后续扩展测试暂缓。

先读取 README.md、synopsys/MEASUREMENT_CONTRACT.md 和现有脚本，执行 verify_upload.py。若已有综合进程运行，先检查当前阶段、日志进展、CPU、内存、swap及已有检查点；不要重复启动第二份综合。正常推进的任务继续复用，确认资源阻塞后先保存日志和可用检查点，再调整流程。

1. 固定测量对象与规模

顶层：attention_head_16ctx_4dim_eval
HEAD_DIM=128
TILE_SIZE=32
CONTEXT_PARALLEL=16
KV_LOAD_PARALLEL=16
MAX_CONTEXT=1152
其余参数保持当前默认值。

这是单 query、单 head FP32 evaluator。
不得缩小参数、修改算法、改变流水线或使用行为模型替代真实DW算子。

本轮只对运行时 N=1152、seed=20261023 采集功耗。
目标是获得 TOTAL、QK/PV MAC、TOTAL−MAC residual 的：

- 周期及延迟 ns

- 标准单元面积 µm²

- 动态、漏电及总能量 pJ
2. 精简功能验证

使用官方 DesignWare FP32 仿真模型。
先验证 N=1、32、33、511、512、513、1152 的数值及周期，再执行必要的复位重用、停顿、背压和非法命令测试。
边界功能测试不采集功耗波形，避免无用波形开销。

暂缓 N=1..1152 全范围扫描、GQA/cohort 回归、其他 N 的能耗测量及另外两个随机种子。
这些项目标为本轮未执行，不影响首组结果单独交付。

默认无停顿 N=1152 预期121拍：
QK=24拍，PV=18拍，调度residual=79拍。
必须以实际仿真日志核对，不直接抄写预期值。

3. 综合与时序

使用 SMIC 28nm：
SCC28NHKCP_HSC35P140_RVT_V0p2
RVT / SS / 0.9V / 125°C，basic NLDM/NLPM。

保持现有1ns及接口约束：
setup/hold uncertainty=50/20ps；
clock/input slew=50ps；
数据输入max/min delay=100/0ps；
输出max/min delay=100/0ps；
每输出负载5fF；
max transition=200ps；
仅rst_n false path，无额外FSM假路径或multicycle。
布局前，无CTS/SPEF。

优先复用与当前源码、库和约束匹配的已有映射结果，不重复综合。

如整体映射确实超出资源，可采用分层映射，复用参数及约束适用的相同模块实现，再完成真实顶层链接与跨层STA。保留16个u_mac边界，禁止未映射黑盒。
不要为了获取单个全局DDC，重复耗时重载；若完整分层网表和顶层链接证据齐全，可据此交付。

完整STA单独执行一次，报告setup/hold、WNS/TNS、违例数量、电气检查、覆盖率及代表性关键路径。
大量违例保留汇总和代表性明细，避免生成数百万行的逐条报告。
本轮不搜索精确Fmax，也不反复扫描时钟周期。

4. 单组活动功耗

仅对N=1152、seed=20261023，采集命令握手至首次o_valid的完整窗口。
排除复位、预热、空闲和结果背压。

功耗阶段复用同一最终映射网表及约束，仍执行必要的update_timing/update_power，但不要重复生成整套STA报告。
仅采集功耗标注所需信号，避免不必要的DW内部调试波形；报告实际活动覆盖率。

TOTAL和16个u_mac必须使用同一网表、同一完整命令窗口。
保留单位核对、活动覆盖率、功耗检查及slew/load超库表范围统计。

5. MAC扣除

面积：
A_residual = A_total − ΣA_u_mac

能量：
E_residual = E_total − ΣE_u_mac
动态、漏电分别相减。

MAC能量包含完整命令窗口内的活动、时钟和漏电，不能只截取MAC活跃阶段。
不得按42/121分摊面积或能量，也不得减去另一份独立综合MAC的结果。

延迟：
C_residual = C_total − C_QK − C_PV
它是调度余项，不能称为独立非MAC关键路径延迟。

1ns未通过时，121/42/79ns仅标为nominal；能耗标为诊断估计，valid延迟和valid能量保留null。
不要求本轮修改RTL解决时序。

6. 可用入口

确认配置与现有产物后，按需执行，避免重复运行：

python3 synopsys/run_flow.py --stage sim --contexts 1152 --seeds 20261023
python3 synopsys/run_flow.py --stage dc
python3 synopsys/run_flow.py --stage sta
python3 synopsys/run_flow.py --stage power
python3 synopsys/collect_candidate.py results/smic28_1ns

上述sim命令用于单组活动；代表性边界及协议测试另行复用编译产物执行。
不要使用未显式限制种子和上下文范围的批量测量命令。

允许调整测量脚本以实现上述精简，保留修改diff；不修改RTL、不更新模拟器或calibration。

7. 交付

生成首组RESULTS_SUMMARY.md和reviewed JSON，列出：
TOTAL / MAC / residual 的周期、nominal_ns、valid_ns、
area_um2、dynamic_pj、leakage_pj、total_pj。

附源码与网表SHA256、库和工具版本、实际约束、功能日志、
面积、STA、电气、活动覆盖及功耗报告、16个MAC实例清单和测量窗口。

缺失项如实标为PENDING。本轮只有一个种子，不报告跨种子的统计结论。
完成首组结果后直接交付，暂不自动扩展测试。



本轮结果分析和模拟器参数统一采用“1周期=1ns”的建模口径。

即使STA未满足1ns约束，也按照功能仿真实测周期数计算：
延迟(ns) = 实测周期数 × 1ns。

不要使用最差路径延迟、WNS或放宽后的时钟周期重新换算本轮建模延迟，也不要凭单算子延迟额外增加周期。

若实测仍为：
TOTAL=121拍、QK=24拍、PV=18拍，则报告：

- Attention总建模延迟：121ns
- QK/PV MAC调度延迟：42ns
- 非MAC调度余项：79ns

周期变化时，以本次实测值重新计算。

主结果表使用上述建模延迟，同时另列STA是否通过、WNS及电气检查结果。明确注明“按1ns/周期建模”，不将其表述为已经实现或通过时序验证的1GHz硬件延迟。

JSON中的nominal_ns填写上述建模值；valid_ns仅在时序检查通过后填写，否则保留null。
面积、能耗继续按实际测量口径报告，不按周期占比推算。
