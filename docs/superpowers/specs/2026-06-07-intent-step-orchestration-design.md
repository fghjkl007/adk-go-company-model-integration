# Intent Step Orchestration Design

## Goal

Design a demo-first, production-shaped architecture for a Go Lambda dialogue orchestrator that supports multiple same-level intents and step-based flows.

The demo should include:

- `MakePaymentIntent`
- `LinkExternalAccountIntent`

All intents are same-level intents. No intent owns another intent. Intents may declare relationships to other intents, and the Orchestrator decides whether to switch, resume, or complete.

This document is architecture only. It does not require implementation code.

## Core Principles

1. Prompt declares what is expected.
2. LLM normalizes user language into a structured `DialogueCommand`.
3. `CommandGuard` validates the command against the current `ExpectedAnswerSpec`, executed step history, and intent relationship metadata.
4. `Orchestrator` routes the command and applies `StepResult`.
5. `Step` owns business handling, but never directly mutates state.
6. `LLM` and ADK callbacks never mutate business state.
7. Resume returns to the original intent and original step, then re-enters that step.
8. The entry intent is accepted only when there is no active conversation. In the current demo it can come from the CLI / test harness; in a future Lex integration it can come from Lex. After `ConversationState.activeIntent` exists, `ConversationState` is the source of truth.

## Top-Level Runtime Path

```text
Demo client / CLI / test harness
  -> entry intent + utterance + session id
  -> Go orchestrator
  -> Orchestrator reads ConversationState from sessionAttributes-equivalent state
  -> RuntimeIntentGuard validates entry intent vs active conversation state
  -> Orchestrator resolves active same-level intent
  -> Step Prompt / ExpectedAnswerSpec
  -> DialogueParser / ADK / LLM returns normalized DialogueCommand
  -> CommandGuard
  -> Orchestrator
  -> Intent / Step
  -> StepResult
  -> Orchestrator updates ConversationState
  -> sessionAttributes-equivalent state
  -> next assistant message
```

The current demo does not connect to Lex. It receives an entry intent from the demo runtime, CLI, test harness, or mock API. The Orchestrator uses that entry intent to start a same-level intent only when there is no active dialogue state.

Once a conversation is already active, the persisted `ConversationState` is the source of truth for `activeIntent` and `activeStep`. The LLM is not the initial intent classifier and is not the flow controller. It only maps natural language into a command shape that Go code can validate and understand.

Mid-flow `switch_intent` is different from the entry intent. It is a structured dialogue command emitted inside an active conversation and must still pass `CommandGuard` plus intent relationship policy. A new external intent label must not directly switch the Orchestrator's active intent.

## Entry Intent Source

The current demo has no Lex `DialogCodeHook`. The entry intent is just an input to the demo runtime.

Current demo shape:

```text
First turn:
  request.entryIntent = MakePayment
  request.utterance = "I want to make a payment"
  Orchestrator initializes ConversationState.activeIntent

Mid-flow turns:
  request.utterance = "change the amount"
  Orchestrator reads ConversationState.activeIntent
  RuntimeIntentGuard ignores any new entryIntent/external intent label
  DialogueParser decides answer / change_step / switch_intent / unknown
```

Future Lex integration note:

```text
First turn:
  Lex classifies entry intent
  Lambda initializes ConversationState.activeIntent

Mid-flow turns:
  Lex still invokes DialogCodeHook every turn
  Lambda reads ConversationState.activeIntent from sessionAttributes
  Lambda ignores any conflicting Lex intent for routing
  DialogueParser decides answer / change_step / switch_intent / unknown
```

Future Lex response guidance:

```text
When waiting for user input:
  return a Lex response that keeps the call in the code-hook path
  keep ConversationState in sessionAttributes
  elicit a generic capture slot or otherwise delegate in a way that returns every user utterance to Lambda
```

When Lex is added later, this can be a single generic slot such as `userUtterance` on the active Lex intent or on a routing/capture intent. The exact Lex wiring can vary, but the invariant is fixed: mid-flow user utterances must come back to Lambda, and Lambda must not trust Lex's new intent classification over `ConversationState.activeIntent`.

### Lex Slot vs Business Slot

The design intentionally separates Lex runtime slots from internal dialogue/business slots.

Lex slot:

```text
userUtterance
```

Purpose:

- Capture the caller's raw speech/text.
- Keep the Lex session routed back to Lambda.
- It can be the same generic slot for every prompt in the demo/future Lex integration.

Internal business slot:

```text
paymentType
paymentAmount
paymentDate
autopayDay
disclosureAccepted
accountType
```

Purpose:

- Tell Go code what business field the LLM normalized.
- Drive `CommandGuard` validation.
- Drive `Step.Handle` business logic.
- Drive disclosure generation and stale-field invalidation.

Important rule:

```text
Lex slot name can be generic and mostly irrelevant.
Internal business slot name must be precise and canonical.
```

So "all questions use one Lex slot" is fine. It does not mean all questions use one internal business slot.

### RuntimeIntentGuard

`RuntimeIntentGuard` runs before `DialogueParser` / `CommandGuard`.

Rules:

```text
if ConversationState has no activeIntent:
  entry intent must map to a registered same-level intent
  initialize activeIntent from entry intent

if ConversationState has activeIntent:
  keep activeIntent from ConversationState
  treat any new entry intent or external intent label as observation only
  if incoming intent label differs from activeIntent:
    log intent_label_mismatch metadata
    do not switch intent
    continue with DialogueParser using ConversationState.activeIntent
```

Only two things can change the active intent after a conversation is active:

```text
DialogueCommand.act = switch_intent
StepResult.transition = delegate_to_intent
```

Both still go through `CommandGuard` / `DelegateIntent` and relationship validation.

## Core Components

### DialogueParser

`DialogueParser` is the adapter around ADK / LLM.

Responsibilities:

- Build the runtime prompt from the latest user utterance, current `ExpectedAnswerSpec`, active intent, active step, LLM-visible step history, allowed switch intents, and safe context.
- Call ADK / LLM.
- Return exactly one normalized `DialogueCommand`.

DialogueParser must not:

- Decide payment business policy.
- Mutate `ConversationState`.
- Read or write Lex `sessionAttributes`.
- Route to the next step by itself.
- See the full step registry or all possible future steps.

### IntentRegistry

Holds all available same-level intents.

Responsibilities:

- Register intent definitions.
- Resolve intent by name.
- Validate whether one intent may route to another.
- Provide relationship metadata for switch/resume behavior.

### Intent

An intent is a same-level flow definition.

Intent-level contract:

```text
Name
EntryStep
Steps
Relationships
ResumePolicy
CompletionPolicy
```

These fields belong to the intent, not to an individual step.

| Field | Meaning |
| --- | --- |
| `Name` | Stable intent name, for example `MakePayment` or `LinkExternalAccount`. |
| `EntryStep` | First step when this intent starts directly. |
| `Steps` | Step registry owned by this intent. This is a Go routing/validation map, not an LLM prompt payload. |
| `Relationships` | Same-level intents this intent may delegate/switch to. This is the source of the LLM-visible allowed switch-intent list. |
| `ResumePolicy` | How this intent should be re-entered when another intent completes and a `ResumeFrame` points back here. |
| `CompletionPolicy` | What should happen when this intent completes. |

Intent responsibilities:

- Own the step graph for that intent.
- Declare same-level intent relationships.
- Declare completion behavior.
- Declare resume behavior.

Intent must not:

- Directly mutate conversation state.
- Call the LLM.
- Apply `StepResult`.

### Step

A step is one question or one decision point inside an intent.

Step contract:

```text
Name
Prepare
Prompt
Handle
InvalidationPolicy
```

Step responsibilities:

- Run business checks for this step.
- Generate the customer-facing prompt.
- Generate the machine-facing `ExpectedAnswerSpec`.
- Handle valid current-step answers.
- Return `StepResult`.

Step must not:

- Directly write `ConversationState`.
- Switch intent by itself.
- Write Lex `sessionAttributes`.

### Step Policies

Only `InvalidationPolicy` lives on the step.

Step navigation is intentionally history-limited. The LLM must not know every step in the active intent. It receives only the active intent's LLM-visible step history: steps that have already been entered/executed and are safe to return to.

The full `StepRegistry` remains code-only. It is used by Go to resolve the step implementation and to validate that a requested step name is real. A `change_step` target must pass both checks:

```text
targetStep exists in active intent StepRegistry
targetStep exists in active intent returnable step history
```

If both checks pass, the current prompt expires and the target step re-enters through `Prepare -> Prompt`.

`InvalidationPolicy` is not an LLM prompt and it does not mutate state by itself. It is read by `Step.Handle` and the Orchestrator.

#### InvalidationPolicy

`InvalidationPolicy` answers: if the answer owned by this step changes, which downstream fields become stale?

It is mainly used after `Step.Handle` accepts a new value and returns `StepResult`.

Example:

```text
Step: getPaymentAmount
User changes amount from 20 to 30
```

Invalidation policy:

```text
When paymentAmount changes:
  clear makePayment.disclosureAccepted
  clear makePayment.disclosureVersion
  clear makePayment.confirmationPreview
```

Reason:

```text
The previous disclosure was read for the old amount.
After the amount changes, disclosure acceptance is no longer valid.
```

Another example:

```text
Step: choosePaymentType
User changes paymentType from onetime to autopay
```

Invalidation policy:

```text
When paymentType changes:
  clear makePayment.paymentDate
  clear makePayment.autopayDay
  clear makePayment.disclosureAccepted
  clear makePayment.disclosureVersion
```

Reason:

```text
One-time payment and autopay use different downstream fields and different disclosure text.
```

#### Policy Ownership

```text
change_step command:
  CommandGuard / Orchestrator checks targetStep exists in active intent StepRegistry
  CommandGuard / Orchestrator checks targetStep exists in active intent returnable step history
  Orchestrator expires current prompt / ExpectedAnswerSpec
  Orchestrator re-enters target step through Prepare -> Prompt

answer command:
  CommandGuard validates slot/value against ExpectedAnswerSpec
  Step.Handle runs step business logic
  Step.Handle returns StepResult
  Orchestrator applies StepResult and uses InvalidationPolicy to clear stale downstream data
```

## Go Design Rules For Intent And Step

These rules are implementation guidance for the Go demo. They keep intent and step code small, testable, and safe in warm Lambda containers.

### Use Typed Names

Use typed string aliases instead of raw strings across the orchestrator.

```go
type IntentName string
type StepName string
type BusinessSlotName string
type CommandAct string
type TransitionType string
```

Rules:

- Define intent, step, and internal business slot names as constants.
- Do not compare magic strings in step code.
- Keep names stable because they are persisted in Lex `sessionAttributes`.
- Do not model Lex's generic capture slot as a business slot.

### Intent Interface Shape

The intent interface should expose metadata and step lookup. It should not execute business state changes.

```go
type Intent interface {
    Name() IntentName
    EntryStep() StepName
    Step(name StepName) (Step, bool)
    Steps() []Step
    Relationships() []IntentRelationship
    ResumePolicy() ResumePolicy
    CompletionPolicy() CompletionPolicy
}
```

Rules:

- `Intent` owns the step registry for that intent.
- `Intent` declares relationships and policies.
- `Intent` does not call ADK / LLM.
- `Intent` does not read or write Lex `sessionAttributes`.
- `Intent` does not mutate `ConversationState`.

### Step Interface Shape

The step interface should expose the enter path, answer handling path, and stale-data policy.

```go
type Step interface {
    Name() StepName

    Prepare(ctx context.Context, state ConversationState) (PrepareResult, error)
    Prompt(ctx context.Context, state ConversationState, prepared PrepareResult) (PromptResult, error)
    Handle(ctx context.Context, state ConversationState, cmd DialogueCommand) (StepResult, error)

    InvalidationPolicy() InvalidationPolicy
}
```

Rules:

- `Prepare` runs when entering or re-entering the step.
- `Prompt` returns the customer message and `ExpectedAnswerSpec`.
- `Handle` runs only after `CommandGuard` accepts a current-step `answer`.
- `Handle` returns `StepResult`; it does not apply it.
- `Step` does not call ADK / LLM.
- `Step` does not read or write Lex `sessionAttributes`.

### Constructor And Dependency Rules

Use constructors and dependency injection. Do not use package-level mutable dependencies.

```go
type MakePaymentIntent struct {
    steps map[StepName]Step
}

type GetPaymentAmountStep struct {
    paymentAPI PaymentAPI
    policy     PaymentPolicy
    base       BaseStep
}
```

`PaymentAPI` and `PaymentPolicy` are placeholder names for injected business dependencies. In a real demo, they can be interfaces such as account lookup, payment eligibility, balance reads, or payment amount validation.

Rules:

- Build the intent registry once during Lambda cold start if useful.
- Treat registry, intent definitions, step definitions, names, and policies as immutable after construction.
- Keep per-customer state out of structs. Per-customer state lives in `ConversationState`.
- Do not store `customerId`, `activeStep`, `ExpectedAnswerSpec`, latest user utterance, or `StepResult` in package variables or long-lived structs.
- Shared clients such as HTTP clients or model adapters may be reused only if they are stateless or thread-safe.

### State Immutability Rule

Step methods receive `ConversationState` as input, but they should not mutate it directly.

Rules:

- `Prepare` may read state and call read APIs.
- `Prompt` may read state and `PrepareResult`.
- `Handle` may read state and return `StepResult`.
- Only the Orchestrator writes updated `ConversationState` back to Lex `sessionAttributes`.
- Prefer explicit `StepResult` updates over hidden mutation.

Example:

```text
Good:
  Handle returns SetIntentData(makePayment.paymentAmount = 20)

Avoid:
  Handle directly changes state.IntentStates.MakePayment.PaymentAmount
```

### Prepare Rules

`Prepare` is for business checks and API reads before asking a question.

Rules:

- Keep `Prepare` idempotent when possible.
- Do not advance the active step in `Prepare`.
- Do not call the LLM in `Prepare`.
- If an API read fails, return an error or a step-specific `PrepareResult` that leads to a safe prompt or transfer policy.

Examples:

```text
checkExternalAccount.Prepare:
  read whether customer has external accounts

getPaymentAmount.Prepare:
  read min/max payment amount or balance if needed
```

### Prompt Rules

`Prompt` converts `PrepareResult` into a customer-facing prompt and machine-facing `ExpectedAnswerSpec`.

Rules:

- `Prompt` should be deterministic from `ConversationState` + `PrepareResult`.
- `Prompt` should not call ADK / LLM.
- `Prompt` must return the `ExpectedAnswerSpec` that will be stored in `sessionAttributes`.
- If the step can skip itself, return that as a transition-like prompt result for the Orchestrator to apply.

Example:

```text
choosePaymentType.Prompt:
  message: "Do you want to make a one-time payment or set up auto pay?"
  expectedSlot: paymentType
  allowedValues: [onetime, autopay]
```

### Handle Rules

`Handle` processes a valid answer to the current step's own prompt.

Rules:

- `Handle` should assume `CommandGuard` already validated command shape and canonical values.
- `Handle` owns step-specific business validation.
- `Handle` may reject an otherwise well-formed answer for business reasons.
- `Handle` returns `StepResult` with updates, clears, optional message, and transition.
- `Handle` does not handle global `change_step`, `switch_intent`, or `unknown`.

Example:

```text
choosePaymentType.Handle:
  value=onetime -> set paymentType=onetime, moveToStep=getPaymentAmount
  value=autopay -> set paymentType=autopay, moveToStep=getPaymentAmount

getPaymentAmount.Handle:
  amount valid and paymentType=onetime -> set paymentAmount, clear disclosure fields, moveToStep=getPaymentDate
  amount valid and paymentType=autopay -> set paymentAmount, clear disclosure fields, moveToStep=getAutopayDay
  amount too high -> stayOnStep with business error prompt

getPaymentDate.Handle:
  date valid -> set paymentDate, clear disclosure fields, moveToStep=readDisclosure

getAutopayDay.Handle:
  day valid -> set autopayDay, clear disclosure fields, moveToStep=readDisclosure
```

`getPaymentDate` and `getAutopayDay` are separate step implementations. They are not one generic "date" step with a conditional write.

Hard write rule:

```text
getPaymentDate.Handle:
  may write makePayment.paymentDate
  must not write makePayment.autopayDay

getAutopayDay.Handle:
  may write makePayment.autopayDay
  must not write makePayment.paymentDate
```

### Policy Implementation Rule

For the demo, prefer static invalidation policy values on the step definition.

```go
type BaseStep struct {
    name               StepName
    invalidationPolicy InvalidationPolicy
}
```

`BaseStep` is an optional helper for shared step metadata. It should not store customer-specific state.

Rules:

- `InvalidationPolicy` should be stable metadata, not ad hoc string logic inside the Orchestrator.
- If a step has custom invalidation, put it near the step implementation so the rule is easy to find.

### Error And Logging Rules

Rules:

- Return Go errors for internal failures.
- Return `StepResult` with `stay_on_step` and a safe message for user-facing business rejections.
- Do not log raw utterance, account number, customer id, amount, date, or full account details.
- Log only metadata such as `trace_id`, `active_intent`, `active_step`, `command_act`, `command_slot`, and `transition_type`.

## Step Lifecycle

Each step has an enter path and an input path.

Only two phase values need to be persisted in Lex `sessionAttributes`:

```text
entering_step
waiting_for_answer
```

`entering_step` means the Orchestrator should run `Prepare -> Prompt`.
`waiting_for_answer` means a prompt has already been sent and the next user utterance should be normalized before routing.

### Enter Phase

Runs when the Orchestrator enters or re-enters a step.

```text
Prepare
  -> Prompt
  -> wait for user input
```

`Prepare` does business checks and API reads. Examples:

- Does the user have an external account?
- Does the user have recent payments?
- Is the customer eligible for this payment type?

`Prompt` uses `Prepare` output to decide what to ask. It also produces `ExpectedAnswerSpec`.

`Prompt` may:

- Ask a question.
- Skip to another step.
- Recommend routing to another intent.
- Reprompt if needed.

### Step Method Result Types

| Type | Produced by | Purpose | Stored where |
| --- | --- | --- | --- |
| `PrepareResult` | `Step.Prepare` | Holds API read results and business checks needed to ask the next question. | Usually request-scoped; optionally summarized in `StepRuntime` while waiting for an answer. |
| `PromptResult` | `Step.Prompt` | Holds the assistant message, `ExpectedAnswerSpec`, and optional skip/route recommendation. | `ExpectedAnswerSpec` and prompt metadata are stored in Lex `sessionAttributes`. |
| `StepResult` | `Step.Handle` | Holds accepted business updates, stale-field clears, optional user-facing message, and transition. | Applied only by the Orchestrator, then written to Lex `sessionAttributes`. |

### Input Phase

Runs after the user answers.

```text
User answer
  -> DialogueCommand
  -> CommandGuard
  -> Orchestrator routes command
  -> Step.Handle only for current-step answer
```

`Handle` only runs for a valid `answer` command that belongs to the current step.

Yes/no is not a separate command act. If the current step expects yes/no, `Step.Prompt` models it as `allowedValues: [yes, no]` for a specific slot such as `disclosureAccepted`.

If the user says something like "link a new account" during an amount step, the command is a global interrupt and does not enter `Handle`.

## Lambda Turn Control Loop

One Lambda invocation should run an internal control loop until it produces exactly one customer-facing response, completes the active intent, or transfers to an agent.

The common happy path is:

```text
phase = waiting_for_answer
  -> parse latest user utterance
  -> CommandGuard
  -> Step.Handle
  -> StepResult(move_to_step)
  -> apply StepResult
  -> phase = entering_step
  -> Prepare
  -> Prompt
  -> save message + ExpectedAnswerSpec
  -> phase = waiting_for_answer
  -> return one Lex response
```

The Orchestrator may enter several steps in the same Lambda invocation if a step skips itself or a `StepResult` immediately moves to another step. The invocation ends when the Orchestrator has a message to say to the customer.

Control loop sketch:

```text
maxTurnIterations = 10

for iteration in 1..maxTurnIterations:
  if phase == waiting_for_answer:
    parse latest user utterance once
    validate DialogueCommand
    route command

    if routing returns a reprompt for the same ExpectedAnswerSpec:
      keep retryCount as incremented
      set phase = waiting_for_answer
      return Lex response

    if routing completes, transfers, or returns a customer-facing message:
      return Lex response

    apply command or StepResult
    continue

  if phase == entering_step:
    run Step.Prepare
    run Step.Prompt

    if Prompt returns a new customer-facing question:
      save ExpectedAnswerSpec
      reset retryCount = 0
      set phase = waiting_for_answer
      return Lex response

    if Prompt skips, moves, delegates, completes, or transfers:
      apply transition
      continue

if maxTurnIterations is exceeded:
  transfer_to_agent or return a safe fallback
```

This loop is the only place that chains multiple deterministic transitions inside one Lambda call. The LLM is not allowed to run an autonomous loop.

### Retry Count

`retryCount` belongs to the current prompt, not the whole intent.

Increment `retryCount` when the current user utterance cannot be accepted for the current prompt:

```text
unknown
low confidence
invalid slot
invalid value
invalid yes/no for the current ExpectedAnswerSpec
```

Retry behavior:

```text
retryCount = 0 when a new prompt is created
retryCount += 1 after each failed attempt for the same prompt
retryCount < 3 -> reprompt current step
retryCount >= 3 -> transfer_to_agent or fallbackPolicy handoff
```

Reset `retryCount` to `0` whenever the prompt changes:

```text
move_to_step
change_step
switch_intent
delegate_to_intent
resume from ResumeFrame
new prompt / new ExpectedAnswerSpec after entering a step
```

Do not reset `retryCount` when returning a reprompt for the same `ExpectedAnswerSpec`.

If the user switches step or intent, the previous prompt is obsolete, so its retry count is also obsolete.

## State Model

`ConversationState` is the structured dialogue state stored in Lex `sessionAttributes` for the demo. Lambda reads it from `sessionAttributes` at the start of each turn and writes the updated version back to `sessionAttributes` in the Lex response.

Production storage can be revisited later, but this demo assumes the whole conversation state is carried through Lex `sessionAttributes`.

```text
ConversationState
  CustomerContext
  IntentStates
  ActivePointer
  SuspensionStack
```

### CustomerContext

Cross-intent customer facts.

Examples:

```text
hasExternalAccount
externalAccounts
hasRecentPayment
recentPayments
balance
eligibilityFlags
```

Rules:

- This is not a Go package global.
- It must be stored in durable conversation state or hydrated from APIs.
- It can be marked stale by `StepResult` updates.
- It should not store raw PII or account numbers.

### IntentState

Intent-specific business draft.

Examples for `MakePaymentIntent`:

```text
paymentType
paymentAmount
paymentDate
autopayDay
paymentMethod
paymentAccountRef
disclosureAccepted
disclosureVersion
confirmationId
```

`autopayDay` is part of `MakePaymentIntent` state even when the current branch is one-time and the value is empty. Keeping the field visible in the state contract prevents the autopay branch from being missed by implementers.

Examples for `LinkExternalAccountIntent`:

```text
accountType
verificationStatus
linkedAccountRef
```

Each intent owns its own `IntentState`.

### StepHistory

`StepHistory` records which steps in an intent have already been entered/executed and are safe to expose back to the LLM as `change_step` targets.

Recommended shape:

```text
stepHistoryByIntent:
  makePayment:
    - choosePaymentAccount
    - choosePaymentType
    - getPaymentAmount
  linkExternalAccount:
    - collectAccountType
```

Rules:

- Store step history in `ConversationState`, carried through session attributes for the demo.
- Keep history per intent. Do not merge `MakePayment` history with `LinkExternalAccount` history.
- Send only the active intent's returnable step history to the LLM.
- Do not send the full `StepRegistry` to the LLM.
- Append a step when it produces a customer-visible prompt or owns an accepted / prefilled value.
- Deduplicate while preserving first-entry order.
- If a step has never been executed or filled, the user cannot `change_step` to it yet.

This keeps the LLM from jumping to future steps that the customer has never reached.

### StepRuntime

Current step runtime data.

Examples:

```text
prepareResult
expectedAnswerSpec
retryCount
awaitingInput
lastPromptType
```

`StepRuntime` helps avoid rerunning `Prepare` while waiting for the user's answer. When a step is resumed after another intent completes, the step re-enters and refreshes `Prepare` / `Prompt`.

If `change_step` or `switch_intent` happens while a step is waiting for an answer, the current prompt is obsolete.

- `change_step` enters the target step with `phase = entering_step`.
- `switch_intent` enters the target intent's entry step with `phase = entering_step`.
- A suspended step must not resume in `waiting_for_answer`; it resumes as `entering_step`.

| Field | Meaning |
| --- | --- |
| `prepareResult` | Optional cached or summarized result from `Prepare`, used only while waiting for the answer to the current prompt. |
| `expectedAnswerSpec` | The machine-readable answer contract generated by `Prompt`. |
| `retryCount` | Number of failed attempts for the current prompt. Reset to `0` whenever the prompt identity changes through move, change step, switch intent, delegate, resume, or a new `ExpectedAnswerSpec`. Do not reset for a same-prompt reprompt. |
| `awaitingInput` | Whether the system is waiting for the user's answer to the current prompt. |
| `lastPromptType` | Optional prompt classification such as normal question, reprompt, disclosure, or fallback. |

### Returnable Step Target Rules

The customer can ask to change an earlier answer, but only earlier answers that exist.

Examples:

```text
If getPaymentAmount has already prompted the user or owns a prefilled paymentAmount:
  LLM-visible change_step targets may include getPaymentAmount

If getPaymentAmount has not been reached and no amount has been accepted/prefilled:
  do not include getPaymentAmount in the LLM prompt
  CommandGuard rejects targetStep=getPaymentAmount if the model invents it
```

This keeps `change_step` aligned with the conversation the customer has actually experienced.

### ActivePointer

Tracks the current active intent and step.

```text
activeIntent
activeStep
phase
```

| Field | Meaning |
| --- | --- |
| `activeIntent` | Current same-level intent controlled by the Orchestrator. |
| `activeStep` | Current step inside `activeIntent`. |
| `phase` | Persisted runtime phase. Use only `entering_step` or `waiting_for_answer`. `handling answer` and `completing intent` are transient code paths, not stored session state. |

### SuspensionStack

Stores explicit resume frames when one same-level intent is interrupted to launch another same-level intent.

```text
ResumeFrame
  intent
  step
  reason
  resumeMode
```

Resume only happens if a `ResumeFrame` exists.

| Field | Meaning |
| --- | --- |
| `intent` | Suspended intent to return to. |
| `step` | Suspended step to re-enter. |
| `reason` | Why the intent was suspended, for example user selected link new account. |
| `resumeMode` | How to resume, usually re-enter original step and rerun `Prepare -> Prompt`. |

`ResumeFrame` intentionally does not preserve the old phase. Returning to a suspended step always sets `ActivePointer.phase = entering_step`.

## Dialogue Contract

### ExpectedAnswerSpec

Generated by `Step.Prompt`.

```text
expectedSlot
allowedValues
confidenceThreshold
repromptPolicy
```

| Field | Meaning |
| --- | --- |
| `expectedSlot` | Internal business slot / business field the current step expects, for example `paymentType`. This is not the Lex capture slot. |
| `allowedValues` | Optional canonical value set, for example `[onetime, autopay]`. |
| `confidenceThreshold` | Minimum parser confidence for accepting a normalized command without reprompt. |
| `repromptPolicy` | How to reprompt after unknown, low confidence, or invalid value. |

Example:

```text
step: choosePaymentType
expectedSlot: paymentType
allowedValues: [onetime, autopay]
```

`ExpectedAnswerSpec` only describes the current step's expected answer shape. It does not decide whether `change_step` or `switch_intent` is allowed. Those global routing commands are validated by `CommandGuard` against the active intent's returnable step history and allowed same-level intent relationships.

### LLM-Visible Routing Context

The LLM prompt should contain only the routing targets that are valid for the current turn.

```text
active_intent
active_step
current ExpectedAnswerSpec
returnable_change_step_targets from active intent StepHistory
allowed_switch_intents from active intent Relationships
```

It must not contain:

```text
full StepRegistry
future unvisited steps
all registered intents
unrelated intent relationships
```

This means `StepRegistry` and `IntentRegistry` are Go-side maps. The LLM sees a filtered, turn-specific view.

### DialogueCommand

Generated by the LLM / ADK parser.

```text
act
slot
value
targetIntent
targetStep
confidence
reason
```

| Field | Meaning |
| --- | --- |
| `act` | What the user is trying to do: `answer`, `change_step`, `switch_intent`, or `unknown`. Yes/no is represented as an `answer` value, not an act. |
| `slot` | Internal business slot being answered when `act=answer`, for example `paymentType`. This is not the Lex capture slot. |
| `value` | Canonical value understood by Go code, for example `autopay`. |
| `targetIntent` | Target intent when `act=switch_intent`. |
| `targetStep` | Target step when `act=change_step`. |
| `confidence` | Parser confidence score. |
| `reason` | Short non-customer-facing explanation for audit/debugging. |

Allowed `act` values:

```text
answer
change_step
switch_intent
unknown
```

Normalization examples:

```text
"automatic pay" -> act=answer, slot=paymentType, value=autopay
"recurring payment" -> act=answer, slot=paymentType, value=autopay
"single payment" -> act=answer, slot=paymentType, value=onetime
"tomorrow" -> act=answer, slot=paymentDate, value=<canonical date>
"the fifteenth" during getAutopayDay -> act=answer, slot=autopayDay, value=15
"yes" during a disclosure step -> act=answer, slot=disclosureAccepted, value=yes
"sure" during a disclosure step -> act=answer, slot=disclosureAccepted, value=yes
"no" during a disclosure step -> act=answer, slot=disclosureAccepted, value=no
```

The exact canonical value must match what existing Go validation expects.

If the current step expects `paymentAmount` and the user only says "yes", the parser should return `unknown` or `CommandGuard` should downgrade it to `unknown`. The word "yes" is accepted only when the current `ExpectedAnswerSpec` explicitly allows `yes`.

## Guards

### RuntimeIntentGuard

`RuntimeIntentGuard` protects the Orchestrator from any incoming entry/external intent label after a conversation is already active.

It validates the runtime request before command parsing:

```text
if no active ConversationState:
  incoming entry intent must map to a registered same-level intent
  initialize activeIntent from that entry intent

if active ConversationState exists:
  ignore any incoming entry intent / external intent label for routing
  keep ConversationState.activeIntent
  keep ConversationState.activeStep
  pass the raw user utterance to DialogueParser

if active ConversationState exists and incoming intent label differs:
  log intent_label_mismatch
  do not mutate state
```

Current demo note: the incoming entry intent can come from CLI, test harness, or mock API. Future Lex note: if Lex later reclassifies a mid-flow utterance, the same guard prevents Lex from hijacking the active Orchestrator state.

### CommandGuard

`CommandGuard` is the final validator before any state mutation.

Validation rules:

```text
act must be one of: answer, change_step, switch_intent, unknown

if act == answer:
  slot must match ExpectedAnswerSpec.expectedSlot
  value must be in ExpectedAnswerSpec.allowedValues when allowedValues is present
  yes/no values are valid only when allowedValues explicitly includes yes/no

if act == switch_intent:
  targetIntent must exist
  targetIntent must be in the active intent's allowed switch-intent relationships

if act == change_step:
  targetStep must exist in the active intent StepRegistry
  targetStep must exist in the active intent returnable step history

if invalid:
  downgrade to unknown
```

Invalid commands never mutate state.

## Orchestrator Routing

The Orchestrator routes by validated `DialogueCommand.act`.

```text
answer
  -> call active Step.Handle

change_step
  -> do not call current Handle
  -> require target step in active intent StepRegistry
  -> require target step in active intent returnable step history
  -> mark the current prompt / ExpectedAnswerSpec obsolete
  -> move to target step
  -> set ActivePointer.phase = entering_step
  -> invalidate downstream IntentState data

switch_intent
  -> do not call current Handle
  -> require target intent in active intent allowed switch-intent list
  -> default resumeBehavior = push_current_step
  -> call DelegateIntent(targetIntent, reason, resumeBehavior)

unknown
  -> increment retryCount
  -> reprompt current step if retryCount < 3
  -> trigger fallbackPolicy / transfer if retryCount >= 3
```

Only `answer` goes to current `Step.Handle`. `change_step`, `switch_intent`, and `unknown` are handled by the Orchestrator before step business logic.

### Unified Intent Delegation

Any intent transfer must use the same Orchestrator helper, regardless of where it came from:

```text
DialogueCommand.act = switch_intent
StepResult.transition = delegate_to_intent
```

Both call:

```text
DelegateIntent(targetIntent, reason, resumeBehavior)
```

Default resume behavior:

```text
DialogueCommand.act = switch_intent during an active conversation
  -> resumeBehavior defaults to push_current_step

Direct entry intent with no active ConversationState
  -> no switch_intent command is involved
  -> no ResumeFrame is pushed
```

This makes global interrupts consistent with the happy-path `delegate_to_intent` case. If the user is already inside `MakePayment` and asks to link an account, the Orchestrator suspends the current MakePayment step by default and resumes it after `LinkExternalAccount` completes.

`DelegateIntent` owns the shared mechanics:

```text
validate same-level intent relationship
mark current prompt / ExpectedAnswerSpec obsolete
reset retryCount = 0
push ResumeFrame(intent, step, reason) when resumeBehavior = push_current_step
activate target intent at target EntryStep
set ActivePointer.phase = entering_step
```

This keeps the happy path and global interrupt path consistent:

```text
Happy path:
  answer -> Step.Handle -> StepResult(delegate_to_intent) -> DelegateIntent(...)

Global interrupt:
  switch_intent command -> DelegateIntent(...)
```

## StepResult

`Step.Handle` returns `StepResult`. It does not apply the result itself.

Suggested shape:

```text
StepResult
  updates
  clears
  transition
  message
```

| Field | Meaning |
| --- | --- |
| `updates` | Business state values to set in `ConversationState`. |
| `clears` | Fields to clear because they are stale or no longer applicable. |
| `transition` | What should happen next: stay, move step, delegate, complete, or transfer. |
| `message` | Optional immediate customer-facing message, usually for stay-on-step business rejection. |

Examples:

```text
SetIntentData(makePayment.paymentAmount = 20)
SetIntentData(makePayment.paymentDate = 2026-06-08)
ClearIntentData(makePayment.paymentAccountRef)
InvalidateIntentData(makePayment.disclosureAccepted)
InvalidateIntentData(makePayment.disclosureVersion)
MarkCustomerContextStale(recentPayments)
MarkCustomerContextStale(balance)
CompleteStep(makePayment.getPaymentAmount)
CompleteIntent(makePayment)
```

| Operation | Meaning |
| --- | --- |
| `SetIntentData` | Set a field in one intent's `IntentState`. |
| `ClearIntentData` | Remove or reset a field in one intent's `IntentState`. |
| `InvalidateIntentData` | Mark a field stale so it must be regenerated or asked again. |
| `SetCustomerContext` | Set a cross-intent customer context fact. |
| `MarkCustomerContextStale` | Mark a customer context fact stale so it must be refreshed from API later. |
| `CompleteStep` | Mark a step complete for audit/progress. |
| `CompleteIntent` | Mark the active intent complete and trigger `CompletionPolicy`. |

Suggested transition types:

```text
stay_on_step
move_to_step
delegate_to_intent
complete_intent
transfer_to_agent
```

| Transition | Meaning |
| --- | --- |
| `stay_on_step` | Stay on current step, usually with a business rejection prompt. |
| `move_to_step` | Move to another step in the same intent. |
| `delegate_to_intent` | Delegate control to another same-level intent through the shared `DelegateIntent` path. |
| `complete_intent` | Finish the active intent and evaluate `CompletionPolicy`. |
| `transfer_to_agent` | Stop self-service flow and transfer to a human agent. |

`delegate_to_intent` should include enough transition data for the Orchestrator to decide resume behavior:

```text
transition:
  type: delegate_to_intent
  targetIntent: LinkExternalAccount
  reason: user_selected_link_new_account
  resumeBehavior: push_current_step
```

Allowed `resumeBehavior` values:

```text
push_current_step
no_resume
```

`push_current_step` means the Orchestrator writes a `ResumeFrame` before activating the target intent. `no_resume` means the target intent completes independently.

### StepResult Apply Order

The Orchestrator applies `StepResult`.

```text
1. Update ConversationState:
   - set new values
   - clear stale values
2. Apply transition:
   - active step
   - delegate via DelegateIntent
   - resume
   - complete intent
   - reset retryCount when prompt/step/intent changes
3. Write updated ConversationState to Lex sessionAttributes.
```

If the transition does not produce a customer-facing response, the Lambda turn control loop continues. For example, after `move_to_step`, the same invocation enters the next step, runs `Prepare -> Prompt`, and returns the next question.

`CommandGuard` already validates command shape and canonical values. `Step.Handle` owns step-specific business logic. The Orchestrator does not re-run those checks; it applies the returned `StepResult` through one common path.

`StepResult` should be auditable and deterministic.

## Intent Policies

### ResumePolicy

`ResumePolicy` answers: if this intent was suspended, how should the Orchestrator return to it?

Example for `MakePayment`:

```text
Mode: resume_original_step
RerunPrepare: true
RerunPrompt: true
```

Meaning:

```text
When LinkExternalAccount completes and a ResumeFrame points back to MakePayment,
delegate back to MakePayment, return to the original step, and rerun Prepare -> Prompt.
```

### CompletionPolicy

`CompletionPolicy` answers: when this intent completes, what should happen next?

Example for `LinkExternalAccount`:

```text
If ResumeFrame exists:
  pop ResumeFrame
  delegate back to the caller intent

If ResumeFrame does not exist:
  finish LinkExternalAccount normally
  return main menu / Lex complete
```

Relationship policy decides whether one intent may delegate to another.
Resume policy decides how a suspended intent is re-entered.
Completion policy decides what happens when the active intent finishes.

## Same-Level Intent Relationship

All intents are same-level intents.

`MakePaymentIntent` may declare a relationship to `LinkExternalAccountIntent`.

`IntentRelationship` is the Go representation of this routing metadata. It tells the Orchestrator which same-level intent routes are allowed and why.

At runtime, the active intent's relationships are filtered into `allowedSwitchIntents` and sent to the LLM prompt. The LLM sees only those peer intents, each with a short description. It does not see every registered intent.

Relationship example:

```text
from: makePayment
to: linkExternalAccount
allowedReasons:
  - missing_external_account
  - user_selected_link_new_account
  - user_wants_new_account
  - replace_payment_account
primaryTrigger:
  step: makePayment.choosePaymentAccount
  prompt: use previous account or link a new account
resumePolicy:
  resume_if_resume_frame_exists
```

| Field | Meaning |
| --- | --- |
| `from` | Source intent where the relationship starts. |
| `to` | Target same-level intent that may be delegated to. |
| `allowedReasons` | Allowed business reasons for this route. |
| `primaryTrigger` | Main happy-path step/prompt that usually causes this relationship. |
| `resumePolicy` | Relationship-specific resume rule; route is allowed, but resume still requires a `ResumeFrame`. |

The relationship allows routing. It does not force resume.

## Resume Rules

Resume behavior depends on `SuspensionStack`.

### Relationship-Launched Intent

```text
MakePayment active
MakePayment asks whether to use previous account or link a new account
User chooses to link a new account
Step.Handle returns StepResult(delegate_to_intent LinkExternalAccount, resumeBehavior=push_current_step)
Orchestrator calls DelegateIntent(...)
DelegateIntent marks the current MakePayment prompt obsolete
DelegateIntent pushes ResumeFrame(makePayment, currentStep, reason)
LinkExternalAccount active
LinkExternalAccount completes
Orchestrator pops ResumeFrame
MakePayment resumes at original step with phase = entering_step
```

### Direct-Launched Intent

```text
No suspended MakePayment
User asks to link an external account
entryIntent = LinkExternalAccount
LinkExternalAccount active
LinkExternalAccount completes
SuspensionStack is empty
Do not resume MakePayment
Finish / main menu / Lex complete
```

Hard rule:

```text
Intent relationship allows a route.
ResumeFrame causes a resume.
No ResumeFrame means no automatic return.
```

### Resume Re-Enter Rule

Resume returns to the original intent and original step, then re-enters the step.

```text
Resume target = original intent + original step
Resume phase = entering_step
Resume behavior = run Prepare -> Prompt again
```

This avoids using stale `StepRuntime` after a linked intent changes `CustomerContext`.

The old user question is considered expired when `change_step` or `switch_intent` happens. The Orchestrator must not reuse the previous `ExpectedAnswerSpec` as if it were still waiting for an answer.

## Global Interrupt Policy

At any step, the user may interrupt.

Examples:

```text
"I want to use another card"
"I want to link an external account"
"I picked checking, but I want to link a new account"
"change the payment amount"
```

Global interrupts are handled before current `Step.Handle`.

This is not the primary happy path for linking an account. The primary happy path is `MakePayment.choosePaymentAccount` asking whether the user wants to use the previous account or link a new account. The global interrupt rule exists so the same relationship still works if the user asks to link an account later in the flow.

Examples:

```text
"I want to use another card"
  -> act=change_step
  -> targetStep=choosePaymentAccount
  -> valid only if choosePaymentAccount is in the active intent returnable step history

"I want to link an external account"
  -> act=switch_intent
  -> targetIntent=linkExternalAccount
  -> valid only if MakePayment allows switch/delegate to LinkExternalAccount

"I picked checking, but I want to link a new account"
  -> act=switch_intent
  -> targetIntent=linkExternalAccount
  -> reason=replace_payment_account
```

If payment instrument changes, downstream data must be invalidated:

```text
ClearIntentData(makePayment.paymentMethod)
ClearIntentData(makePayment.paymentAccountRef)
InvalidateIntentData(makePayment.disclosureAccepted)
InvalidateIntentData(makePayment.disclosureVersion)
```

## Demo Intent Flows

### MakePaymentIntent

Suggested steps:

```text
checkExternalAccount
choosePaymentAccount
choosePaymentType
getPaymentAmount
getPaymentDate
getAutopayDay
readDisclosure
```

Branching:

```text
choosePaymentType = onetime -> getPaymentAmount -> getPaymentDate -> readDisclosure
choosePaymentType = autopay -> getPaymentAmount -> getAutopayDay -> readDisclosure
```

MakePayment uses two different date-like slots:

| Payment type | Step | Slot | Canonical value | Example utterance |
| --- | --- | --- | --- | --- |
| `onetime` | `getPaymentDate` | `paymentDate` | Full calendar date, for example `2026-06-10` | "tomorrow", "June tenth", "today" |
| `autopay` | `getAutopayDay` | `autopayDay` | Day of month integer `1` through `31` | "on the fifteenth", "the 1st of each month", "every month on day 20" |

Rules:

- Do not reuse `paymentDate` for autopay.
- Do not reuse `autopayDay` for one-time payment.
- Do not collapse `getPaymentDate` and `getAutopayDay` into one generic step.
- `getPaymentDate` writes only `makePayment.paymentDate`.
- `getAutopayDay` writes only `makePayment.autopayDay`.
- `CommandGuard` validates `paymentDate` as a canonical date only when `ExpectedAnswerSpec.expectedSlot = paymentDate`.
- `CommandGuard` validates `autopayDay` as an integer from `1` to `31` only when `ExpectedAnswerSpec.expectedSlot = autopayDay`.
- Disclosure generation reads `paymentDate` for one-time payment and `autopayDay` for autopay, because the disclosure wording is different.

If user changes amount, date, payment type, or payment account, disclosure data becomes stale.

### LinkExternalAccountIntent

Suggested steps:

```text
collectAccountType
collectAccountDetails
verifyAccount
confirmLinkedAccount
```

Completion effects:

```text
SetCustomerContext(hasExternalAccount = true)
MarkCustomerContextStale(externalAccounts)
SetIntentData(linkExternalAccount.linkedAccountRef = ...)
CompleteIntent(linkExternalAccount)
```

After completion:

```text
if SuspensionStack has ResumeFrame:
  resume original intent and step
  set ActivePointer.phase = entering_step
  re-enter Prepare -> Prompt
else:
  finish LinkExternalAccount normally
```

## Error Handling

`fallbackPolicy` means how the Orchestrator responds when the command is unknown, low confidence, or invalid. Examples: reprompt once, reprompt with choices, or transfer after too many failed attempts.

`transferPolicy` means when the self-service flow should stop and transfer to a human agent. Examples: API failure, repeated unknown input, unsupported payment request, or compliance-sensitive path.

Failure behavior:

```text
LLM no tool call -> unknown
low confidence -> unknown or reprompt
invalid slot -> unknown
invalid value -> unknown
invalid target intent -> unknown
invalid target step -> unknown
parser error -> unknown
state save failure -> do not advance state
API failure in Prepare -> step-specific error prompt or transfer policy
```

The customer-facing fallback is owned by the Orchestrator / step policy, not by the LLM.

## Logging And Privacy

Logs should include:

```text
trace_id
active_intent
active_step
command_act
command_slot
confidence
next_action
step_result_ops
```

`step_result_ops` means the operation names contained in the applied `StepResult`, for example `SetIntentData`, `ClearIntentData`, or `CompleteIntent`. Do not log the sensitive values carried by those operations.

Logs must not include:

```text
raw user utterance
amount
date
account number
customer id
full account details
```

ADK callbacks can log trace metadata and perform early shape checks, but they must not mutate payment state.

In this document, ADK callbacks means before/after agent, model, or tool hooks used for tracing, metrics, and shape checks. They are not business workflow callbacks.

## Testing

Required demo tests:

1. MakePayment happy path.
2. LinkExternalAccount direct path completes without resume.
3. MakePayment -> LinkExternalAccount -> resumes original intent and original step.
4. Resume re-enters the step and reruns Prepare -> Prompt.
5. Change step and switch intent expire the current prompt and continue with `phase=entering_step`, never `waiting_for_answer`.
6. User says "automatic pay"; parser normalizes to canonical `autopay`.
7. User says "single payment"; parser normalizes to canonical `onetime`.
8. User changes amount after later steps; disclosure is invalidated.
9. User asks to link account from any step; global interrupt switches intent.
10. Invalid LLM value is downgraded by `CommandGuard`.
11. During disclosure, user says "yes" / "sure" / "no problem"; parser returns `act=answer`, `slot=disclosureAccepted`, `value=yes`.
12. During amount collection, user says "yes"; `CommandGuard` downgrades it to `unknown` because `paymentAmount` does not allow yes/no.
13. One Lambda invocation can handle `StepResult(move_to_step)`, enter the next step, run `Prepare -> Prompt`, and return exactly one next question.
14. Unknown / invalid input increments `retryCount`; after 3 failed attempts, fallback / transfer is triggered.
15. `change_step`, `switch_intent`, `delegate_to_intent`, and resume reset `retryCount` to `0`.
16. `StepResult(delegate_to_intent)` and `DialogueCommand.switch_intent` both use `DelegateIntent`, so both can push `ResumeFrame` when `resumeBehavior=push_current_step`.
17. Mid-flow incoming intent mismatch is ignored: with `ConversationState.activeIntent=MakePayment`, if a new incoming/external intent label conflicts with the active state, Lambda keeps MakePayment and lets DialogueParser produce the appropriate command.
18. `change_step` to a step in active intent `StepRegistry` but not in active intent `StepHistory` is downgraded to `unknown`.
19. `switch_intent` to an intent that exists globally but is not in the active intent's allowed switch-intent relationships is downgraded to `unknown`.
20. `switch_intent` emitted during an active conversation defaults to `resumeBehavior=push_current_step`, so global interrupt to `LinkExternalAccount` resumes the original `MakePayment` step after link completion.
21. Sequential Lambda calls do not share state except persisted `ConversationState`.

Optional tests:

- Concurrent parser calls do not share captured command or session state.

## Acceptance Criteria

The design is successful when:

- Adding a new intent does not require rewriting the Orchestrator.
- Adding a new step does not require changing unrelated steps.
- LLM output is always normalized and validated before state mutation.
- Entry intent source is used only when no active `ConversationState` exists. After `ConversationState.activeIntent` exists, new incoming/external intent labels are not allowed to replace Orchestrator state.
- `change_step` can target only steps that exist in active intent `StepRegistry` and active intent `StepHistory`.
- The full `StepRegistry` is never sent to the LLM.
- `switch_intent` can target only peer intents declared by the active intent's relationship list.
- MakePayment can route to LinkExternalAccount and resume only when a ResumeFrame exists.
- Direct LinkExternalAccount completion does not resume MakePayment.
- Resumed intents return to the original intent and original step, then re-enter the step.
- `change_step` and `switch_intent` expire the current prompt; target/resumed steps always use `phase=entering_step` and rerun `Prepare -> Prompt`.
- Step business logic returns `StepResult`; only the Orchestrator applies it.
- A Lambda turn returns at most one customer-facing message and has `maxTurnIterations = 10` loop protection.
- `retryCount` is scoped to the current prompt, maxes at 3 failed attempts, and resets when prompt/step/intent changes.
- `switch_intent` commands and `delegate_to_intent` StepResults share the same `DelegateIntent` path, including ResumeFrame behavior.
- `switch_intent` defaults to `resumeBehavior=push_current_step` when emitted during an active conversation.
- Mid-flow intent changes are allowed only through `DialogueCommand.switch_intent` or `StepResult.delegate_to_intent`, never directly from a new external intent label.
- CustomerContext, IntentState, and StepRuntime have clear boundaries.
- The only command acts are `answer`, `change_step`, `switch_intent`, and `unknown`; yes/no responses are canonical `answer` values validated against the current `ExpectedAnswerSpec`.
