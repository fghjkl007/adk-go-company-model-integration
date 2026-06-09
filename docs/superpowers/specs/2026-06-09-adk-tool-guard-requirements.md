# ADK Tool And Guard Requirements

## Purpose

This requirement document defines how the demo should use Google ADK tools and callbacks for the intent / step orchestration demo.

The goal is not to let the LLM execute payment logic. The goal is to make the LLM produce one structured, validated `DialogueCommand` through a controlled ADK tool call.

## Core Requirement

Use one shared ADK tool for all intents and steps:

```text
emit_dialogue_command
```

All intents and steps use the same tool. The active step changes the prompt context, not the tool registry.

The LLM must call this tool to express what the user is trying to do:

```text
answer
change_step
switch_intent
unknown
```

The LLM must not directly mutate conversation state, write Lex session attributes, call payment APIs, create accounts, or move the state machine.

## Required ADK APIs

The implementation should use the ADK Go tool and callback surface.

Required ADK concepts / methods:

```go
functiontool.New(...)
llmagent.New(llmagent.Config{...})
Tools: []tool.Tool{emitDialogueCommandTool}
BeforeModelCallbacks: []llmagent.BeforeModelCallback{...}
AfterModelCallbacks: []llmagent.AfterModelCallback{...}
BeforeToolCallbacks: []llmagent.BeforeToolCallback{...}
AfterToolCallbacks: []llmagent.AfterToolCallback{...}
runner.New(...)
session.InMemoryService()
```

`session.InMemoryService()` is request-scoped for the demo parser call. It is not the customer conversation store. Customer conversation state remains in the orchestrator state / session attributes.

## Tool Contract

Tool name:

```text
emit_dialogue_command
```

Tool purpose:

```text
Convert the latest user utterance into exactly one DialogueCommand.
```

Suggested tool args:

```text
act
slot
value
targetIntent
targetStep
confidence
reason
```

Allowed `act` values:

```text
answer
change_step
switch_intent
unknown
```

Important: `slot` means internal business slot, not Lex slot.

Examples:

```text
paymentType
paymentAmount
paymentDate
autopayDay
disclosureAccepted
accountType
```

Lex can use one generic capture slot such as `userUtterance`. The business slot returned by the tool must remain precise.

## Agent Construction Requirement

The parser should build an ADK `LlmAgent` with the shared command tool and callbacks.

Shape:

```go
emitTool, err := functiontool.New(
    functiontool.Config{
        Name:        "emit_dialogue_command",
        Description: "Emit one normalized dialogue command for the current turn.",
    },
    emitDialogueCommand,
)

agent, err := llmagent.New(llmagent.Config{
    Name:        "dialogue_command_parser",
    Model:       model,
    Instruction: instruction,
    Tools:       []tool.Tool{emitTool},

    BeforeModelCallbacks: []llmagent.BeforeModelCallback{beforeModel},
    AfterModelCallbacks:  []llmagent.AfterModelCallback{afterModel},
    BeforeToolCallbacks:  []llmagent.BeforeToolCallback{beforeTool},
    AfterToolCallbacks:   []llmagent.AfterToolCallback{afterTool},
})
```

This snippet is a contract shape, not a mandate to copy exact function names. The implementation should follow the ADK Go API available in the installed ADK version.

## Prompt Context Requirement

Each parser call must send only the current turn's safe routing context to the LLM.

Allowed context:

```text
activeIntent
activeStep
lastAssistantQuestion
ExpectedAnswerSpec
ReturnableStepHistory for active intent only
AllowedSwitchIntents for active intent only
safe CustomerContext summary
latest user utterance
```

Do not send:

```text
full StepRegistry
future unvisited steps
all registered intents
raw account numbers
full customer identity
payment execution credentials
complete audit logs
```

## Tool Handler Requirement

The `emit_dialogue_command` handler is only a capture adapter.

It may:

```text
receive typed tool args
convert args into DialogueCommand
store the command in request-scoped capture
return a small success marker to ADK
```

It must not:

```text
update ConversationState
write session attributes
call payment APIs
call account-link APIs
decide next step
push ResumeFrame
invalidate disclosure
transfer to agent
```

The Orchestrator owns all state mutation.

## Guard Layers

The implementation must have two guard layers after the ADK tool call.

### 1. ADK Tool-Level Guard

Use `BeforeToolCallbacks` and / or the tool handler to validate the tool call shape.

This layer checks:

```text
tool name must be emit_dialogue_command
args must parse
act must be one of the allowed command acts
confidence must be 0.0 through 1.0 if present
required fields must exist for the selected act
logs must not contain PII
```

If invalid, block the tool execution or return an error-shaped tool result that causes parser output to become `unknown`.

This guard does not apply business state changes.

### 2. Go CommandGuard

After ADK returns a captured `DialogueCommand`, the Go parser must pass it to the existing `CommandGuard`.

This layer checks current conversation validity:

```text
if act = answer:
  slot must match ExpectedAnswerSpec.expectedSlot
  value must match ExpectedAnswerSpec.allowedValues when present
  paymentDate must be a canonical date when expectedSlot = paymentDate
  autopayDay must be an integer 1 through 31 when expectedSlot = autopayDay

if act = change_step:
  targetStep must exist in active intent StepRegistry
  targetStep must exist in active intent StepHistory

if act = switch_intent:
  targetIntent must exist
  targetIntent must be in active intent AllowedSwitchIntents
  resumeBehavior defaults to push_current_step during active conversation
```

If invalid, downgrade to `unknown` and let the Orchestrator reprompt / retry / transfer.

## Callback Requirements

### BeforeModelCallbacks

Use for request-level model safety checks.

Responsibilities:

```text
verify model request has one user message
verify instruction is present
verify parser context is bounded
log trace metadata only
do not log raw utterance or sensitive values
```

### AfterModelCallbacks

Use for model response telemetry.

Responsibilities:

```text
log finish reason if available
log token usage if available
log whether any tool call was observed
do not choose fallback message
do not mutate state
```

### BeforeToolCallbacks

Use for tool allowlist and argument shape checks.

Responsibilities:

```text
allow only emit_dialogue_command
reject or block unknown tool names
validate high-level args shape
log tool name, act, slot, targetIntent, targetStep, confidence
never log amount/date/account values in raw form
```

### AfterToolCallbacks

Use for audit-only tool execution result logging.

Responsibilities:

```text
log success/failure
log command act metadata
log captured = true/false
do not modify captured command
do not mutate ConversationState
```

## Parser Return Requirement

The parser should return exactly one `DialogueCommand`.

If ADK does not call `emit_dialogue_command`, return:

```text
act = unknown
reason = no_emit_dialogue_command
```

If ADK calls the wrong tool, return:

```text
act = unknown
reason = invalid_tool_call
```

If ADK emits invalid args, return:

```text
act = unknown
reason = invalid_tool_args
```

The Orchestrator decides the customer-facing fallback.

## Example Commands

Current step:

```text
activeIntent = MakePayment
activeStep = choosePaymentType
ExpectedAnswerSpec.expectedSlot = paymentType
ExpectedAnswerSpec.allowedValues = [onetime, autopay]
```

User says:

```text
automatic payment
```

Tool args:

```json
{
  "act": "answer",
  "slot": "paymentType",
  "value": "autopay",
  "confidence": 0.92
}
```

Current step:

```text
activeIntent = MakePayment
activeStep = getAutopayDay
ExpectedAnswerSpec.expectedSlot = autopayDay
```

User says:

```text
on the fifteenth
```

Tool args:

```json
{
  "act": "answer",
  "slot": "autopayDay",
  "value": 15,
  "confidence": 0.91
}
```

Current step:

```text
activeIntent = MakePayment
activeStep = readDisclosure
AllowedSwitchIntents = [LinkExternalAccount]
```

User says:

```text
I want to link a new account
```

Tool args:

```json
{
  "act": "switch_intent",
  "targetIntent": "LinkExternalAccount",
  "reason": "user_wants_new_account",
  "confidence": 0.9
}
```

Go routing:

```text
CommandGuard validates targetIntent
Orchestrator calls DelegateIntent(...)
resumeBehavior defaults to push_current_step
ResumeFrame is pushed
LinkExternalAccount starts at EntryStep
```

## Non-Goals

Do not add these ADK tools for the demo:

```text
make_payment
create_payment
link_external_account
lookup_customer_accounts
write_session_attributes
transfer_to_agent
```

Those actions belong to Go steps, injected business APIs, policy services, and the Orchestrator.

## Acceptance Criteria

- The demo has exactly one ADK command-emitting tool: `emit_dialogue_command`.
- Every intent and step uses the same tool.
- The LLM never receives the full `StepRegistry`.
- The LLM receives only active intent `StepHistory` and active intent `AllowedSwitchIntents`.
- `BeforeToolCallbacks` allow only `emit_dialogue_command`.
- Tool args pass tool-level shape validation before being captured.
- Captured command still goes through Go `CommandGuard`.
- Invalid / missing / wrong tool calls become `unknown`.
- Tool handler and callbacks never mutate `ConversationState`.
- Only the Orchestrator applies `StepResult` and writes session attributes.
- Direct entry intent does not push `ResumeFrame`.
- Active-conversation `switch_intent` defaults to `resumeBehavior=push_current_step`.
