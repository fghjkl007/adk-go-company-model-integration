# 紧凑版代码包 prompt（用于截图传递）

在已生成的 `adk-code-pack.md` 基础上，生成 `adk-code-pack-lite.md`。目标：截图张数最少，且每屏信息密度最高。

## 格式（紧凑型）
1. 全文只用**一个** ```go 围栏；各段之间用一行 `// ==== 路径 ====` 分隔，不用 markdown 标题、不用空行。
2. 删掉所有注释、空行、import 块、package 行、日志语句（log./logger./writerLogger 开头的整条语句）。
3. 2 空格缩进，行宽不超过 100 列。struct 字段一行一个、不做列对齐；单行函数合并成一行；`if err != nil { return x, err }` 写成一行。
4. 长字符串常量（指令模板、提示文案）用 `"<instruction v3, N lines>"` 占位，不抄正文。
5. 核实清单每条一行，格式：`// Q1 是|否|不确定 文件名:函数名 一句证据`。
6. 文件第一行写 `// total N lines`。

## 内容与顺序（按优先级，P0 放最前）
P0：
- 核实清单 8 条（上一轮第五部分）。
- go.mod 中 adk / genai / openai / aws-sdk lambda 的行；分支名；最近 3 条 commit 的一行摘要。
- adk/parser：`Parser` 接口；`ParseInput`、`PrefillInput`、`ContextPack` struct；`ADKParser` 的 `Parse` 与 `runAgentTurn`（或等价函数）函数体；emit 工具注册代码与 `adkCommandInput` struct。
- adk/adkbridge：`LambdaModel` struct、构造函数、`Generate`/`generate` 函数体、信封 struct。
- adk/dialogue/command.go 的 `DialogueCommand` struct 与 `Unknown`；adk/state/step_runtime.go 的 `StepRuntime` struct。
P1：
- adk/orchestrator：`commandForTurn`、`handleUnknown`、`unknownAnswerTransferReason`、`runPrefillSubPhase` 函数体；`handleAnswerPhase` 只要按 act 分发的 switch 段。
- `NewADKParserFromEnv`；MockParser 的 struct 与方法。
P2：
- `ADKParser.Prefill`、`validateToolArgs`、`validateConfidencePresent` 函数体。
- llmlba/handler.go 中 `Handle` 里创建超时 ctx 与调用 `lexadapter.Run` 的几行。
- `openai_llm.go` 的函数签名列表。

完成后报告总行数，以及 P0 / P1 / P2 各自的行数。
