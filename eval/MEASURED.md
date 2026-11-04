# 实测数字（2026-08-22）

命令：`HF_HUB_OFFLINE=1 python -m eval.runner --task all --llm`  
模型：DeepSeek `deepseek-chat`（API 上的 DeepSeek-V3）  
精排：本地 Cross-Encoder `BAAI/bge-reranker-base`（已加载）  
召回稠密路：本地 `BAAI/bge-small-zh-v1.5`（512 维，本轮 JSON 的 `dense_backend`）

> 本轮报告的标识是 `all_20260822T121308Z.json`。`eval/reports/*.json` 已在 `.gitignore` 中忽略、不随仓库分发，所以仓库里看不到报告本体；本文数字即该轮口径，复现命令与依赖版本见上。

上一轮 hash 对照（同日更早，报告标识 `all_20260822T042748Z.json`，同样不入库）：仅向量 Hit@1 **3.1%**。本轮已换成 small，不再把 Hybrid 当成 hash。

## 结论一览

- 100 条意图：关键词规则 51% → 无覆盖 `intent_v1` 85% / `intent_v2` 86% → `intent_v3`+歧义覆盖 **100%**
- 65 条政策检索 / 39 篇文档：BM25 **Precision@1 76.9%**、**Recall@3 90.0%**、MRR **0.84**
- 仅 `bge-small-zh-v1.5`：**P@1 83.1%**、**R@3 94.6%**、MRR **0.89**
- BM25 + small + RRF：与仅 small 相同（**83.1 / 94.6 / 0.89**）
- Hybrid + Cross-Encoder：**P@1 89.2%**、**R@3 97.7%**、MRR **0.94**（相对纯 BM25 +12.3 / +7.7 个百分点）
- 生成 65 条（同模型裁判）：无检索 Faithfulness **80%** / 幻觉 **20%** → RAG **96.9%** / **3.1%** → RAG+自纠错 **100%** / **0%**；相对下降 **100%**；RAG 关键词 **95.4%**
- E2E 16 条：拒答 100%、工具路径 100%；关键词命中 **9/11（81.8%）**；P50 **4.9s** / P95 **5.8s**（n=16 离散分位；最慢 e02 11.1s）
- pytest：**154 passed**（2026-10-10 下午复测；新增的是 `eval` 指标与检索核心路径（cache / self_rag / query_rewrite / vector / tokenizer）的单测，不改动任何业务数字）

## 口径限制

- E2E 关键词命中是 9/11（81.8%），不是 100%。
- 意图 100% 含覆盖层，不代表纯 LLM 达到 100%。
- RRF 在本 65 条上不比单独使用 small 更好，增益来自 Cross-Encoder。
- 稠密路是 `bge-small-zh-v1.5`，不是 bge-m3。

## 生成 / 幻觉

65 条 `--task generation --llm`（2026-08-22 12:13Z）：无检索 Faithfulness 80%、幻觉 20%、关键词 36.9%；RAG 96.9% / 3.1% / 95.4%；RAG+自纠错 100% / 0% / 95.4%。相对下降 100%。同模型裁判偏乐观。

## 其他需要说明的限制

- 意图 100% 含覆盖层：DeepSeek 先分类，再对转人工/政策 vs 退换货等高混淆短语纠偏。覆盖层在 100 条金标上单独也是 100%。
- 稠密路是 **bge-small-zh-v1.5**（~90MB），不是 bge-m3。
- 同日 hash 对照 Hit@1 仅 3.1%；接上 small 后仅向量 P@1 已到 83.1%。
- 国内需 `HF_ENDPOINT=https://hf-mirror.com` 才能拉 CE / small 权重；本轮离线加载缓存。
