# Biomedical Text-to-SQL 项目：分步实施指南

基于 proposal *Retrieval-Augmented Biomedical Text-to-SQL with Parameter-Efficient Fine-Tuning*（COSI 115b）整理的可执行步骤。

---

## 项目目标（一句话）

构建端到端系统：自然语言问题 → 检索相关 schema → 生成 SQL → 执行 → 返回结果；并在 **RAG 有无**、**多种训练方式**（提示词 / 全量微调 / LoRA·QLoRA）下系统对比。

---

## 阶段 0：环境与仓库

1. 固定 Python 版本与依赖（建议 `requirements.txt` 或 `pyproject.toml`）：`transformers`、`peft`、`datasets`、`sentence-transformers`、`faiss-cpu`（或 GPU 版）、`torch`、`gradio` 等。
2. 建立清晰目录：数据脚本、训练脚本、RAG、评估、demo。
3. 准备 GPU 环境说明（本地 / Colab / 学校集群），便于复现实验。

---

## 第 1 周（Apr 1–7）：代码与数据管线 + 基线 + 生医 schema 起步

| 步骤 | 内容 |
|------|------|
| 1.1 | 下载并预处理 **Spider**（10,181 问 / 200 库）：统一成训练/验证/测试划分与模型输入格式（如 `question` + `schema` → `sql`）。 |
| 1.2 | 实现 **零样本基线**：将 gold schema（完整表结构）拼进 prompt，生成 SQL（上界：schema 选择无误差）。 |
| 1.3 | 实现 **少样本基线**：3–5 个 in-context 示例 + schema，同一生成流程。 |
| 1.4 | 搭建最小 **执行与评估**：对生成 SQL 做语法检查、在对应库上执行，计算 **execution accuracy**；可选 **exact-match**。 |
| 1.5 | **开始构造小型生医数据集**（约 15–20 个 schema，100–150 问答对）：基因表达、临床、药物相互作用等；schema 可参考 MIMIC-III、UniProt 等公开设计文档。 |
| 1.6 | 为后续 RAG 预留：表/列 **自然语言描述** 字段，便于后面做 embedding。 |

---

## 第 2 周（Apr 8–14）：T5 全量微调 + RAG 管线 + 基线评估

| 步骤 | 内容 |
|------|------|
| 2.1 | 在 Spider 上对 **T5-base** 做 **全量 fine-tuning**（输入：问题 + schema/context；输出：SQL）。 |
| 2.2 | 用 **sentence-transformer** 对所有表/列描述做向量；建立 **FAISS** 索引。 |
| 2.3 | 实现 **检索**：用户问题编码 → top-k 表/列 → 将检索到的 schema 片段 **prepend** 到 prompt。 |
| 2.4 | 跑通 **无 RAG** 的微调模型与 **有 RAG** 的推理对比（同一评估脚本）。 |
| 2.5 | 记录 Week 2 **基线数字**（execution / exact-match），为后续消融做参照。 |

---

## 第 3 周（Apr 15–21）：LoRA / QLoRA 消融 + T5 vs CodeT5 + RAG 消融

| 步骤 | 内容 |
|------|------|
| 3.1 | 用 **HuggingFace PEFT** 实现 **LoRA** 与 **QLoRA** 微调（主模型仍可用 T5-base）。 |
| 3.2 | **消融实验**（建议固定若干组合后再扫次要超参）：**rank** 4 / 8 / 16 / 32；**learning rate**；**target modules**（如 attention 投影层等）。 |
| 3.3 | 引入 **CodeT5-base**，在相同数据与评估协议下与 **T5-base** 对比（可同样接 LoRA/QLoRA）。 |
| 3.4 | 在所有训练设定下，系统对比 **with RAG vs without RAG**（同一 checkpoint、同一 k、公平 prompt）。 |
| 3.5 | 按 Spider 难度分层（easy / medium / hard / extra-hard）做 **错误分析** 与简单统计。 |

---

## 第 4 周（Apr 22–28）：生医域评估 + 终实验 + Gradio Demo

| 步骤 | 内容 |
|------|------|
| 4.1 | 在自建 **生医小数据集** 上跑选定的若干代表模型（不必穷举所有消融，但要有说服力）。 |
| 4.2 | 补齐遗漏实验、固定随机种子、保存 **日志与 checkpoint** 命名规范。 |
| 4.3 | 实现 **Gradio**：用户输入自然语言 → 展示生成 SQL + 执行结果（若需可限制只读库或 mock）。 |
| 4.4 | **开始报告**：方法、实验设置、主要表格与图。 |

---

## 第 5 周（Apr 29–May 6）：报告、代码仓库与答辩

| 步骤 | 内容 |
|------|------|
| 5.1 | 完稿 written report；README（如何复现数据、训练、评估、启动 demo）。 |
| 5.2 | 代码整理：脚本入口、配置项、实验记录（可选 wandb / 简单 CSV）。 |
| 5.3 | 制作 **presentation slides**；**May 6** 答辩。 |

---

## 评估指标（贯穿各周）

- **Execution accuracy**：生成 SQL 执行结果是否与 gold 一致（主指标）。
- **Exact-match accuracy**：字符串级匹配（辅助）。
- **分层分析**：按 SQL 难度与（可选）是否使用 RAG 分层。

---

## 核心对照实验矩阵（proposal 要求）

| 维度 | 内容 |
|------|------|
| 训练方式 | 零样本 / 少样本 → 全量 FT → LoRA / QLoRA（含 rank、LR、target modules 消融） |
| 基座模型 | T5-base vs CodeT5-base |
| RAG | with vs without schema retrieval |

---

## 依赖的公开资源（备忘）

- **Spider**：Yale，text-to-SQL 主基准。
- **生医扩展**：MIMIC-III 文档、UniProt 等公开 schema 设计作参考。

---

*文档由 proposal 拆解为可执行步骤；执行时可根据实际进度在每周内微调顺序，但建议保持「先管线与指标，再大规模扫实验」的顺序以降低返工。*
