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
3. `CommandGuard` validates the command against the current `ExpectedAnswerSpec` and the intent registry.
4. `Orchestrator` routes the command and applies validated `StateEffects`.
5. `Step` owns business handling, but never directly mutates state.
6. `LLM` and ADK callbacks never mutate business state.
7. Resume returns to the original intent and original step, then re-enters that step.
8. Lex remains the first-layer intent classifier. Lambda receives a Lex-classified intent before invoking the dialogue orchestrator.

## Top-Level Runtime Path

```text
Amazon Connect / Lex
  -> Lex classified intent + utterance + session id
  -> Go Lambda
  -> Orchestrator reads ConversationState from Lex sessionAttributes
  -> Orchestrator resolves active same-level intent
  -> Step Prompt / ExpectedAnswerSpec
  -> DialogueParser / ADK / LLM returns normalized DialogueCommand
  -> CommandGuard
  -> Orchestrator
  -> Intent / Step
  -> StepResult
  -> StateEffects
  -> EffectApplier
  -> Lex sessionAttributes
  -> next assistant message
```

Lex is still responsible for first-layer intent classification. The Orchestrator uses the Lex intent to start or route the top-level same-level intent when there is no active suspended dialogue state.

Once a conversation is already active, the persisted `ConversationState` is the source of truth for `activeIntent` and `activeStep`. The LLM is not the initial intent classifier and is not the flow controller. It only maps natural language into a command shape that Go code can validate and understand.

Mid-flow `switch_intent` is different from initial Lex classification. It is a structured dialogue command emitted inside an active conversation and must still pass `CommandGuard` plus intent relationship policy.

## Core Components

### IntentRegistry

Holds all available same-level intents.

Responsibilities:

- Register intent definitions.
- Resolve intent by name.
- Validate whether one intent may route to another.
- Provide relationship metadata for switch/resume behavior.

### Intent

An intent is a same-level flow definition.

Intent contract:

```text
Name
EntryStep
Steps
Relationships
ResumePolicy
CompletionPolicy
```

Intent responsibilities:

- Own the step graph for that intent.
- Declare same-level intent relationships.
- Declare completion behavior.
- Declare resume behavior.

Intent must not:

- Directly mutate conversation state.
- Call the LLM.
- Apply state effects.

### Step

A step is one question or one decision point inside an intent.

Step contract:

```text
Name
Prepare
Prompt
Handle
ChangePolicy
InvalidationPolicy
```

Step responsibilities:

- Run business checks for this step.
- Generate the customer-facing prompt.
- Generate the machine-facing `ExpectedAnswerSpec`.
- Handle valid current-step answers.
- Return `StateEffects` and `NextAction`.

Step must not:

- Directly write `ConversationState`.
- Switch intent by itself.
- Write Lex `sessionAttributes`.

## Step Lifecycle

Each step has two phases.

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

### Input Phase

Runs after the user answers.

```text
User answer
  -> DialogueCommand
  -> CommandGuard
  -> Step.Handle
```

`Handle` only runs for valid `answer`, `confirm`, or `deny` commands that belong to the current step.

If the user says something like "link a new account" during an amount step, the command is a global interrupt and does not enter `Handle`.

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
- It can be marked stale by state effects.
- It should not store raw PII or account numbers.

### IntentState

Intent-specific business draft.

Examples for `MakePaymentIntent`:

```text
paymentType
paymentAmount
paymentDate
paymentMethod
paymentAccountRef
disclosureAccepted
disclosureVersion
confirmationId
```

Examples for `LinkExternalAccountIntent`:

```text
accountType
verificationStatus
linkedAccountRef
```

Each intent owns its own `IntentState`.

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

### ActivePointer

Tracks the current active intent and step.

```text
activeIntent
activeStep
phase
```

### SuspensionStack

Stores explicit resume frames when one same-level intent is interrupted to launch another same-level intent.

```text
ResumeFrame
  intent
  step
  phase
  reason
  resumeMode
```

Resume only happens if a `ResumeFrame` exists.

## Dialogue Contract

### ExpectedAnswerSpec

Generated by `Step.Prompt`.

```text
expectedSlot
allowedValues
allowedActs
confidenceThreshold
repromptPolicy
```

Example:

```text
step: choosePaymentType
expectedSlot: paymentType
allowedValues: [onetime, autopay]
allowedActs: [answer, change_step, switch_intent, unknown]
```

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

Allowed `act` values:

```text
answer
confirm
deny
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
```

The exact canonical value must match what existing Go validation expects.

## Command Guard

`CommandGuard` is the final validator before any state mutation.

Validation rules:

```text
if act == answer:
  slot must match ExpectedAnswerSpec.expectedSlot
  value must be in ExpectedAnswerSpec.allowedValues when allowedValues is present

if act == confirm or deny:
  current step must allow confirm/deny

if act == switch_intent:
  targetIntent must exist
  active intent must allow relationship to targetIntent

if act == change_step:
  targetStep must exist
  targetStep must be allowed by ChangePolicy

if invalid:
  downgrade to unknown
```

Invalid commands never mutate state.

## Orchestrator Routing

The Orchestrator routes by validated `DialogueCommand.act`.

```text
answer / confirm / deny
  -> call active Step.Handle

change_step
  -> do not call current Handle
  -> move to target step
  -> invalidate downstream IntentData

switch_intent
  -> do not call current Handle
  -> validate same-level intent relationship
  -> push ResumeFrame if switching away from an active intent
  -> activate target intent

unknown
  -> reprompt current step or trigger fallback policy
```

Only `answer`, `confirm`, and `deny` go to current `Step.Handle`.

## State Effects

`Step.Handle` returns state effects. It does not apply them.

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

### Effect Apply Order

The Orchestrator applies effects through an `EffectApplier`.

```text
1. Validate effect is allowed.
2. Apply data changes.
3. Apply invalidation.
4. Apply transition.
5. Persist new ConversationState.
```

Effects should be auditable and deterministic.

## Same-Level Intent Relationship

All intents are same-level intents.

`MakePaymentIntent` may declare a relationship to `LinkExternalAccountIntent`.

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

The relationship allows routing. It does not force resume.

## Resume Rules

Resume behavior depends on `SuspensionStack`.

### Relationship-Launched Intent

```text
MakePayment active
MakePayment asks whether to use previous account or link a new account
User chooses to link a new account
Orchestrator pushes ResumeFrame(makePayment, currentStep)
LinkExternalAccount active
LinkExternalAccount completes
Orchestrator pops ResumeFrame
MakePayment resumes at original step
```

### Direct-Launched Intent

```text
No suspended MakePayment
User asks to link an external account
Lex intent = LinkExternalAccount
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
Resume behavior = run Prepare -> Prompt again
```

This avoids using stale `StepRuntime` after a linked intent changes `CustomerContext`.

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

"I want to link an external account"
  -> act=switch_intent
  -> targetIntent=linkExternalAccount

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
  resume original intent and step, re-enter Prepare -> Prompt
else:
  finish LinkExternalAccount normally
```

## Error Handling

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
effect_types
```

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

## Testing

Required demo tests:

1. MakePayment happy path.
2. LinkExternalAccount direct path completes without resume.
3. MakePayment -> LinkExternalAccount -> resumes original intent and original step.
4. Resume re-enters the step and reruns Prepare -> Prompt.
5. User says "automatic pay"; parser normalizes to canonical `autopay`.
6. User says "single payment"; parser normalizes to canonical `onetime`.
7. User changes amount after later steps; disclosure is invalidated.
8. User asks to link account from any step; global interrupt switches intent.
9. Invalid LLM value is downgraded by `CommandGuard`.
10. Sequential Lambda calls do not share state except persisted `ConversationState`.

Optional tests:

- Concurrent parser calls do not share captured command or session state.

## Acceptance Criteria

The design is successful when:

- Adding a new intent does not require rewriting the Orchestrator.
- Adding a new step does not require changing unrelated steps.
- LLM output is always normalized and validated before state mutation.
- MakePayment can route to LinkExternalAccount and resume only when a ResumeFrame exists.
- Direct LinkExternalAccount completion does not resume MakePayment.
- Resumed intents return to the original intent and original step, then re-enter the step.
- Step business logic returns `StateEffects`; only the Orchestrator applies them.
- CustomerContext, IntentState, and StepRuntime have clear boundaries.
