# ADK 引擎代码包整理 prompt

在 `cookie-lex-lambda` 仓库根目录、`feature-DARE-37487` 分支上，把下面整段贴给 Claude Code。

---

请在当前仓库整理一份「ADK 引擎代码包」，输出为单个文件 `adk-code-pack.md`，供另一位架构评审者离线阅读。要求：原文照抄代码，不要总结、不要改写、不要省略函数体；每个文件一节，标题格式 `### 路径 (N 行)`，正文用 ```go 围栏。脱敏：AWS 账号 ID、ARN、网关主机名、密钥、testdata 里任何像真实客户数据的内容一律替换为 <REDACTED>，Lambda 函数名和 env 变量名保留。

## 第一部分：环境
1. `git rev-parse --abbrev-ref HEAD`、`git log --oneline -8`、`git status --short`（说明 10-19 的 Answered / prompt v3 改动是否已提交）。
2. `go.mod` 里 google.golang.org/adk、google.golang.org/genai、openai（任何 openai 客户端库）、aws-sdk-go-v2 lambda 相关的行。
3. `go list -deps ./codehook/adk/parser/... ./codehook/adk/adkbridge/... | grep -E 'adk|genai|openai'` 的结果。

## 第二部分：要深化的模块（整文件照抄）
- `codehook/adk/parser/` 下所有非测试 .go 文件（parser.go、adk_parser.go、adk_tool.go、adk_callbacks.go、context_pack.go、adk_instruction.go、mock 实现、其他）。
- `codehook/adk/parser/testdata/` 下的 .golden 文件。
- `codehook/adk/adkbridge/` 下所有非测试 .go 文件。
- `codehook/adk/dialogue/command.go`。
- `codehook/adk/state/step_runtime.go`。

## 第三部分：调用方（只抄相关函数，带函数签名和完整函数体）
- `codehook/adk/orchestrator/orchestrator.go`：RunTurn、loop、handleAnswerPhase、commandForTurn、handleUnknown、unknownAnswerTransferReason、maxRetriesForActiveStep。
- `codehook/adk/orchestrator/prefill.go`：runPrefillSubPhase 整个文件。
- `codehook/llmlba/handler.go`：Handle 函数和 //lex:handler 构造函数，以及 parseTimeout 等常量。
- `codehook/adk/lexadapter/run.go`：Run 函数和 Deps 结构体。
- parser 的冷启动构造：NewADKParserFromEnv 及其读取的 env 变量和默认值。

## 第四部分：测试与外部
- `go test -list . ./codehook/adk/parser/... ./codehook/adk/adkbridge/... ./codehook/adk/orchestrator/...` 的输出（只要测试名）。
- `adkacceptancetests/` 里任意一个使用 MockParser 的测试文件整文件照抄，加上 MockParser 的类型定义所在文件。
- 如果仓库或同一工作区里有 lex-llm-invocation-lambda 的 Python 源码：照抄 handler 主文件；没有就写「不在本仓库」。

## 第五部分：核实清单（每条用「是 / 否 / 不确定」加一行证据，引用文件名和函数名）
1. adkbridge 的 generate 只把 Model、Messages、Tools 放进请求，temperature 和 tool_choice 没有传。
2. ADKParser.Parse 与 Prefill 在任何情况下都返回 nil error。
3. 工具触发后通过 cancel ctx 加 break 退出 runner 循环。
4. emit 工具注册时没有传 InputSchema。
5. handler 的 7 秒 ctx 超时最终表现为 Unknown 而不是 HandleError 转人工。
6. LLM_REQ / LLM_RESP 日志通过直写 stderr 的 logger 发出，绕过 logging2.ConfigureSecureLogging。
7. LEX_LLM_LAMBDA_NAME / GSAI_MODEL 未设置时默认指向 UAT 的 lambda 和 gpt-4.1。
8. MockParser 能否脚本化「这一轮调用失败」（而不只是脚本化返回的命令）。

文件如果超过 6000 行，按上面五个部分拆成 adk-code-pack-1.md 和 adk-code-pack-2.md。完成后报告总行数和脱敏了哪些内容。
