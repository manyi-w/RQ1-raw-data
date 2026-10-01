# RQ2：53 个 Django 实例的效果与成本审计

53 个不同实例全部完成官方评测；37 个通过，16 个未通过；resolve rate = 37/53 = 69.81132075%。

## 口径

主表每个实例取最终送评补丁对应的一次生成，保留失败/超时但已有补丁的真实状态；分母均为 53。这不是首次尝试通过率，也不是多次采样的 pass@k。补充表统计同一批实例所有留存重跑请求。

主表对应 1,487 个唯一 API 请求。共审计 121 个 trace、1,800 个唯一请求，OpenRouter 元数据覆盖率 100%；逐请求费用与账单一致，53 个最终会话汇总也全部一致。

Input 指未缓存输入；cache write/read 是其余输入类别，三者互斥。Output 一栏排除 reasoning；模型/API 的 completion/output 汇总本来包含 reasoning，不能再次相加计费。

## 正确结果：最终送评生成

| 项目 | 总 tokens | 平均 tokens / 实例 | 总费用 USD | 平均费用 USD / 实例 |
|---|---:|---:|---:|---:|
| Input（未缓存） | 12,363 | 233.2642 | 0.037089000 | 0.000699792 |
| Cache write（5 分钟） | 1,972,542 | 37,217.7736 | 7.397032500 | 0.139566651 |
| Cache read | 76,123,730 | 1,436,296.7925 | 22.837119000 | 0.430889038 |
| Output（不含 reasoning） | 379,330 | 7,157.1698 | 5.689950000 | 0.107357547 |
| Reasoning | 257,645 | 4,861.2264 | 3.864675000 | 0.072918396 |
| **合计** | **78,745,610** | **1,485,766.2264** | **39.825865500** | **0.751431425** |

- Input 含全部缓存：平均 1,473,747.8302 tokens。
- Output 含 reasoning：平均 12,018.3962 tokens。
- 平均生成时间：283.2264 秒；不包括镜像构建和官方评测。
- 平均 terminal num_turns：29.2830；平均 API 请求：28.0566。
- 生成进程 success=true：50/53；另外 3 个非零退出仍有补丁并完成评测，不能据退出状态判定 resolved。

## 所有留存尝试：含失败、重跑

| 项目 | 总 tokens | 平均 tokens / 原始实例 | 总费用 USD | 平均费用 USD / 原始实例 |
|---|---:|---:|---:|---:|
| Input（未缓存） | 14,940 | 281.8868 | 0.044820000 | 0.000845660 |
| Cache write（5 分钟） | 2,335,154 | 44,059.5094 | 8.756827500 | 0.165223160 |
| Cache read | 89,604,410 | 1,690,649.2453 | 26.881323000 | 0.507194774 |
| Output（不含 reasoning） | 441,614 | 8,332.3396 | 6.624210000 | 0.124985094 |
| Reasoning | 301,026 | 5,679.7358 | 4.515390000 | 0.085196038 |
| **合计** | **92,697,144** | **1,749,002.7170** | **46.822570500** | **0.883444726** |

相比最终生成，额外留存尝试消耗 313 个请求、$6.996705000。这里按 53 个实例摊销，不是每次尝试均值。无有效模型响应的失败可能为零费用；未留存请求不在账内。

## 原 RQ2 脚本的原样输出（仅用于复核）

原代码未修改；在 paper_project 中复制相同代码、建立指向这 53 个最终实例的目录视图，执行 `python analysis/RQ2_effectiveness_cost/analyze_rq2.py`。

原脚本给出平均 input 567、output 162、total 730 tokens，平均 turns 68.5849，时间 283.2264 秒，通过率 37.0%。这些 token 和通过率不能作为正确实验结果。

原因：①分母写死为 100；②逐条累加 assistant 流式片段，同一请求多次出现；③片段的 output_tokens 常是初始占位数而非最终输出；④完全漏掉 cache write/read；⑤仅有一个模式时，脚本仍固定输出 Run-Free 最划算的结论，该结论无对照实验支持。

原样输出：[data_rq2.md](paper_project/analysis/RQ2_effectiveness_cost/data_rq2.md)。

## 可用于论文的结果描述

在 SWE-bench Verified 的 53 个 Django 实例子集上，Claude Code（Claude Sonnet 4.5）采用 run_free 模式，最终补丁解决 37 个实例（69.81%）。以每个实例最终送评补丁对应的生成会话计，平均模型调用费用为 $0.751431，平均生成时间为 283.23 秒。平均未缓存输入、缓存写入、缓存读取、非 reasoning 输出和 reasoning 分别为 233.26、37217.77、1436296.79、7157.17 和 4861.23 tokens。若计入同批实例所有已留存的失败尝试及重跑，平均模型费用为 $0.883445/实例。本结果仅覆盖该 Django 子集；由于缺少 run_full 对照，不报告相对成本节省或模式优劣。

## 价格与审计资料

单位价格（USD / 百万 tokens）：未缓存输入 3；5 分钟 cache write 3.75；1 小时 write 6；cache read 0.30；output（含 reasoning）15。本次写入均为 5 分钟；逐请求原生 prompt 均不超过 200k。

- [Anthropic 官方价格](https://platform.claude.com/docs/en/about-claude/pricing)
- [OpenRouter 缓存计费](https://openrouter.ai/docs/guides/best-practices/prompt-caching)
- [OpenRouter 请求用量元数据](https://openrouter.ai/docs/api/api-reference/generations/get-generation)

费用仅为模型推理费用，不包括镜像构建、评测 CPU、VPN、税费、充值手续费和未留存的独立连通性测试。

机器可读文件：summary.json、per_instance.csv、per_request.csv、per_trace.csv、selected_manifest.json、pricing_basis.json、provider_records/。per_trace.csv 可能包含归档副本，不应直接相加；费用汇总始终按 request_id 去重。

复现命令：

```bash
.venv/bin/python scripts/audit_verified53_costs.py --prepare
.venv/bin/python scripts/audit_verified53_costs.py --fetch --workers 6
.venv/bin/python scripts/audit_verified53_costs.py --report
```
