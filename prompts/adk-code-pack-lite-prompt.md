# 精简版代码包 prompt（用于截图传递）

在已生成的 `adk-code-pack.md` 基础上，再生成一份 `adk-code-pack-lite.md`，目标是截图张数最少。

1. 删掉所有注释行（`//` 与 `/* */`）、空行、import 块、package 行，以及日志调用语句（log./logger./writerLogger 开头的整条语句）。
2. 只保留以下内容，其余文件整个不要：
   - go.mod 中 adk / genai / openai / aws-sdk lambda 的行；当前分支名和最近 3 条 commit。
   - adk/parser：`Parser` 接口；`ParseInput`、`PrefillInput`、`ContextPack` 的 struct；`ADKParser` 的 `Parse`、`Prefill`、`runAgentTurn`（或等价函数）函数体；emit 工具的注册代码与 `adkCommandInput` struct；`validateToolArgs`、`validateConfidencePresent`；`NewADKParserFromEnv`；MockParser 的 struct 与方法。
   - adk/adkbridge：`LambdaModel` 的 struct、构造函数、`Generate` / `generate` 函数体、信封 struct；`openai_llm.go` 只要函数签名列表。
   - adk/orchestrator：`commandForTurn`、`handleUnknown`、`unknownAnswerTransferReason`、`runPrefillSubPhase` 函数体；`handleAnswerPhase` 只要按 act 分发的 switch 段。
   - adk/state/step_runtime.go 的 `StepRuntime` struct；adk/dialogue/command.go 的 `DialogueCommand` struct 与 `Unknown` 函数。
   - llmlba/handler.go 中 `Handle` 里创建超时 ctx 与调用 `lexadapter.Run` 的那几行。
3. 第五部分「核实清单」8 条答复保留，每条一行。
4. 每个代码段前只放一行路径标题；2 空格缩进，行宽不超过 100 列。
5. 完成后报告总行数。
