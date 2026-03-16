# MEMORY.md - Long-Term Memory

## User
- 吴少敏，男，渥太华，加拿大
- 公司做 Steel Detailing，主工具：**Tekla Structures**
- 核心目标：尽量自动化工作流程

## My Identity
- 还未正式命名（用户可能在叫我"小二"？待确认）
- Model: anthropic/claude-sonnet-4-6

## Notes
- 2026-03-15: 首次对话，建立基本信息
- 2026-03-15: 配置完成 — Telegram bot、Ollama qwen3.5:27b 备用模型、PDF_Data_Extractor 子代理（Gemini）
- 2026-03-15: clone 了 TSOpenAPIExamples (branch: 2024) 到 workspace-master
- **下次任务：** 用 Tekla Open API 建立轴线系统（Grid Lines），然后拆分任务给多个子代理完成

## 子代理配置

### 代理一：PDF_Data_Extractor
- **模型：** `google/gemini-2.0-flash`
- **Workspace：** `workspace-pdf-extractor`
- **职责：** 解析 PDF 图纸，提取轴线/标高/截面规格，输出 XML 文件
- **输出目录：** `pipeline/extracted/`
- **SOUL.md：** `workspace-pdf-extractor/SOUL.md` ✅

### 代理二：Model_Generator
- **模型：** `ollama/qwen3.5:27b`
- **Workspace：** `workspace-model-generator`
- **职责：** 读取 XML → Python 调用 Tekla API → 建轴线网格 + 柱 + 梁
- **绝对纪律：** 只建主干框架，禁止生成连接节点/螺栓/焊缝；XML 缺数据必须报错停止
- **日志目录：** `pipeline/logs/`
- **SOUL.md：** `workspace-model-generator/SOUL.md` ✅

### 待建代理
- **下游节点代理** — 负责连接节点、螺栓、焊缝（明天配置）

## 流水线架构
PDF → PDF_Data_Extractor → XML → Model_Generator → Tekla 主体模型 → 节点代理 → 完整模型
