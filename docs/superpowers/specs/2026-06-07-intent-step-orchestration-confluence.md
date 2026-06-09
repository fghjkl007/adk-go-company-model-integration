# Intent / Step Orchestration Demo

## Purpose

Small demo design for showing how a Go Lambda can manage multi-step intents after an entry intent has been provided by the demo runtime. Future Lex integration can provide that first entry intent, but the current demo does not depend on Lex.

Key idea:

```text
The demo runtime provides the entry intent only when no conversation is active.
Mid-flow incoming intent labels are ignored for routing once ConversationState.activeIntent exists.
LLM normalizes the user's answer into a DialogueCommand.
Go Orchestrator reads Lex SessionAttributes, validates/routes, updates ConversationState, and writes updated SessionAttributes.
```

## Demo Scope

- Entry intent comes from the demo runtime / CLI / test harness. Future Lex can provide the same entry intent.
- After `ConversationState.activeIntent` exists, `ConversationState` is the source of truth.
- Demo intents: `MakePayment` and `LinkExternalAccount`.
- Intents are same-level; one intent does not own another.
- `MakePayment` can route to `LinkExternalAccount`.
- Resume happens only when a `ResumeFrame` exists.
- On resume, return to the original intent and original step, then rerun `Prepare -> Prompt`.
- Demo `ConversationState` is stored in Lex `sessionAttributes`.

## Diagram 1: Runtime Architecture

```mermaid
flowchart TD
  User["User utterance"] --> Entry["Demo runtime provides entry intent when needed"]
  Entry --> Lambda["Go Lambda receives entry intent + utterance + session id"]
  Lambda --> SessionRead["Read Lex SessionAttributes"]
  SessionRead --> RuntimeGuard["RuntimeIntentGuard"]
  RuntimeGuard --> Orchestrator["Orchestrator"]
  Orchestrator --> Active["Resolve active intent + active step"]
  Active --> Spec["Current Step + ExpectedAnswerSpec + StepHistory + AllowedSwitchIntents"]
  Spec --> Command["ADK / LLM returns normalized DialogueCommand"]
  Command --> Guard["CommandGuard"]
  Guard --> Route["Orchestrator routes command"]
  Route --> Step["Step.Handle if current-step answer"]
  Step --> StepResult["StepResult"]
  StepResult --> SessionWrite["Orchestrator updates Lex SessionAttributes"]
  SessionWrite --> Response["Next assistant message"]

  Guard -. "invalid" .-> Unknown["Unknown / reprompt"]
  Unknown --> Response
```

Mid-flow rule: if `ConversationState.activeIntent` exists, `RuntimeIntentGuard` ignores conflicting incoming intent labels. Only `DialogueCommand.switch_intent` or `StepResult.delegate_to_intent` can change the active intent.

## Diagram 2: Step Lifecycle

```mermaid
flowchart TD
  Enter["Enter step"] --> Prepare["Prepare: API reads / business checks"]
  Prepare --> Prompt["Prompt: message + ExpectedAnswerSpec"]
  Prompt --> Wait["Wait for user input"]
  Wait --> Parse["ADK / LLM returns normalized DialogueCommand"]
  Parse --> Guard["CommandGuard"]
  Guard --> Act{"Command act"}
  Act -->|answer| Handle["Step.Handle"]
  Act -->|change_step / switch_intent| Route["Orchestrator routing"]
  Act -->|unknown| Reprompt["Reprompt / fallback"]
  Handle --> Result["StepResult"]
  Result --> Apply["Orchestrator writes updated SessionAttributes"]
  Route --> Enter
  Reprompt --> Wait
```

## Diagram 2.5: One Lambda Turn

```mermaid
flowchart TD
  Start["Lambda invocation starts"] --> Phase{"phase"}
  Phase -->|waiting_for_answer| Parse["Parse + Guard"]
  Parse --> Route["Route command"]
  Route --> Handle["Step.Handle if answer"]
  Handle --> Apply["Apply StepResult"]
  Route -->|change/switch/unknown| Apply
  Apply --> Next{"customer message ready?"}
  Next -->|no| Phase
  Next -->|yes| Return["Return one Lex response"]
  Phase -->|entering_step| Enter["Prepare -> Prompt"]
  Enter --> Next
  Apply -. "maxTurnIterations exceeded" .-> Transfer["Safe fallback / transfer"]
```

One Lambda invocation may move through several deterministic steps, but it returns at most one customer-facing message.

## Step Registry vs Step History

| Item | Used by | Purpose |
| --- | --- | --- |
| `StepRegistry` | Go only | Full map of all step implementations inside an intent. Used to resolve code and validate real step names. |
| `StepHistory` | Go + LLM prompt | Steps already executed / filled for the active intent. Used as the only `change_step` target list shown to the LLM. |
| `AllowedSwitchIntents` | Go + LLM prompt | Peer intents the active intent is allowed to delegate/switch to, with short descriptions. |

Rule: the LLM does not see all steps. It only sees current prompt expectations, already returnable steps, and allowed peer intents.

## Lex Slot vs Business Slot

| Concept | Example | Meaning |
| --- | --- | --- |
| Lex capture slot | `userUtterance` | Generic slot used to capture the caller's raw answer and send every turn back to Lambda. This can be the same for every prompt. |
| Internal business slot | `paymentType`, `paymentDate`, `autopayDay`, `disclosureAccepted` | Canonical field returned by ADK / LLM and validated by Go. This must be precise. |

Rule: one generic Lex slot is fine. Internal business slots still stay separate because Go validation, step handling, disclosure text, and stale-field invalidation depend on them.

## Runtime Step Prompt Example

Example step:

| Field | Value |
| --- | --- |
| Intent | `MakePayment` |
| Step | `choosePaymentType` |
| Assistant question | "Do you want to make a one-time payment or set up auto pay?" |
| Expected answer | `paymentType = onetime | autopay` |

Prompt generated by the current Step:

```text
You are a dialogue command normalizer for a phone payment flow.

Your job is to convert the user's latest utterance into one structured DialogueCommand.

You are not the flow controller.
You do not decide business policy.
You do not update state.
You do not invent new intents, steps, slots, or values.

Current context:
- active_intent: MakePayment
- active_step: choosePaymentType
- last_assistant_question: "Do you want to make a one-time payment or set up auto pay?"

ExpectedAnswerSpec:
- if act = answer:
  - slot: paymentType
  - allowed_canonical_values:
    - onetime
    - autopay

Canonical value mapping examples:
- "one time", "one-time", "pay once", "single payment" => onetime
- "auto pay", "automatic payment", "recurring payment", "monthly automatic" => autopay

Allowed switch intents:
- LinkExternalAccount
  - description: User wants to link, add, or use a new external bank account before continuing payment.

Returnable change_step targets already executed in active MakePayment history:
- choosePaymentAccount
  - description: User wants to change which account/payment method to use.

Do not infer or invent other MakePayment steps.
The full StepRegistry is code-only and is not provided to you.

Output contract:
Return exactly one DialogueCommand.

Valid command acts:
- answer
- change_step
- switch_intent
- unknown

Rules:
1. If the user answers the current question, return act=answer with the canonical value.
2. If the user says "automatic payment", "recurring", or similar, normalize it to autopay.
3. If the user says "pay once" or similar, normalize it to onetime.
4. If the user wants to link a new account, return act=switch_intent and target_intent=LinkExternalAccount.
5. If the user wants to change an earlier payment field, return act=change_step only when the target is listed in returnable_change_step targets.
6. If the utterance is unrelated, ambiguous, or unsafe to interpret, return act=unknown.
7. Yes/no is not a separate act. If this step expects yes/no, return act=answer with value=yes or value=no.
8. Do not include explanations.
9. Do not ask the user a question.
10. Do not include values outside the allowed canonical values.

Latest user utterance:
"{USER_UTTERANCE}"
```

Output example: user says "automatic payment".

```json
{
  "act": "answer",
  "intent": "MakePayment",
  "step": "choosePaymentType",
  "slot": "paymentType",
  "value": "autopay",
  "confidence": 0.92
}
```

Output example: user says "I want to link a new account".

```json
{
  "act": "switch_intent",
  "intent": "MakePayment",
  "step": "choosePaymentType",
  "target_intent": "LinkExternalAccount",
  "reason": "User wants to link a new external account before continuing payment.",
  "confidence": 0.9
}
```

Responsibility boundary:

| Component | Responsibility |
| --- | --- |
| Step | Generates the prompt context and `ExpectedAnswerSpec`. |
| ADK / LLM | Normalizes the user's utterance into one `DialogueCommand`. |
| CommandGuard | Validates command shape, canonical values, step history, and allowed switch intents. |
| Step.Handle | Runs business logic and returns `StepResult`. |
| Orchestrator | Updates Lex `SessionAttributes` and handles active step / delegate / resume. |

## Diagram 3: MakePayment To LinkExternalAccount

```mermaid
sequenceDiagram
  participant User
  participant Runtime as Demo Runtime
  participant Orchestrator
  participant MakePayment
  participant LinkExternalAccount
  participant SessionAttributes as Lex SessionAttributes

  User->>Runtime: "I want to make a payment"
  Runtime->>Orchestrator: entryIntent = MakePayment
  Orchestrator->>MakePayment: Start / continue MakePayment
  MakePayment->>User: "Use your previous account or link a new account?"
  User->>Orchestrator: "link a new account"
  Orchestrator->>MakePayment: Step.Handle returns delegate_to_intent
  Orchestrator->>SessionAttributes: DelegateIntent expires prompt + writes ResumeFrame
  Orchestrator->>LinkExternalAccount: Activate LinkExternalAccount with phase=entering_step
  LinkExternalAccount->>Orchestrator: CompleteIntent + StepResult
  Orchestrator->>SessionAttributes: Apply StepResult + pop ResumeFrame
  Orchestrator->>MakePayment: Delegate to MakePayment with phase=entering_step
  MakePayment->>MakePayment: Rerun Prepare -> Prompt
  Orchestrator->>User: Ask refreshed MakePayment question
```

## Diagram 4: Direct LinkExternalAccount

```mermaid
sequenceDiagram
  participant User
  participant Runtime as Demo Runtime
  participant Orchestrator
  participant LinkExternalAccount
  participant SessionAttributes as Lex SessionAttributes

  User->>Runtime: "I want to link an account"
  Runtime->>Orchestrator: entryIntent = LinkExternalAccount
  Orchestrator->>LinkExternalAccount: Start direct LinkExternalAccount
  LinkExternalAccount->>Orchestrator: CompleteIntent + StepResult
  Orchestrator->>SessionAttributes: Apply StepResult
  Orchestrator->>SessionAttributes: Check SuspensionStack
  SessionAttributes-->>Orchestrator: Empty
  Orchestrator->>User: Finish / main menu / Lex complete
```

## Diagram 5: ConversationState In SessionAttributes

```mermaid
flowchart TD
  SessionAttrs["Lex sessionAttributes"] --> Conv["ConversationState JSON"]
  Conv --> Active["ActivePointer"]
  Conv --> Customer["CustomerContext"]
  Conv --> Intents["IntentStates"]
  Conv --> History["StepHistory"]
  Conv --> Stack["SuspensionStack"]

  Active --> ActiveEx["Example: activeIntent=makePayment, activeStep=getPaymentDate"]

  Customer --> CustomerEx["Example: hasExternalAccount=true, balance=120.50"]

  Intents --> MP["MakePayment IntentState"]
  MP --> MPEx["Example fields: paymentType, amount, paymentDate, autopayDay, paymentAccountRef"]

  Intents --> LEA["LinkExternalAccount IntentState"]
  LEA --> LEAEx["Example: accountType=checking, verificationStatus=verified"]

  History --> HistEx["Example: makePayment=[choosePaymentAccount, choosePaymentType, getPaymentAmount]"]

  Stack --> Frame["ResumeFrame"]
  Frame --> FrameEx["Example: makePayment.choosePaymentAccount"]
```

## Diagram 6: SessionAttributes While Linking Account

```mermaid
flowchart TD
  SessionAttrs["Lex sessionAttributes while LinkExternalAccount is active"] --> Conv["ConversationState JSON"]
  Conv --> Active["ActivePointer"]
  Conv --> Intents["IntentStates"]
  Conv --> Customer["CustomerContext"]
  Conv --> History["StepHistory"]
  Conv --> Stack["SuspensionStack"]

  Active --> ActiveValue["Example: activeIntent=linkExternalAccount, activeStep=collectAccountType"]

  Intents --> MP["MakePayment IntentState"]
  MP --> MPEx["Example fields retained: paymentType, amount, paymentDate, autopayDay, paymentAccountRef"]

  Intents --> LEA["LinkExternalAccount IntentState"]
  LEA --> LEAEx["Example: accountType=pending, verificationStatus=pending"]

  Customer --> Ctx["Example: externalAccounts=[previousAccount], balance=120.50"]

  History --> HistEx["Example: makePayment=[choosePaymentAccount], linkExternalAccount=[collectAccountType]"]

  Stack --> Frame["ResumeFrame"]
  Frame --> FrameEx["Example: makePayment.choosePaymentAccount"]
  FrameEx --> Resume["On completion: resume makePayment.choosePaymentAccount and rerun Prepare -> Prompt"]
```

## Diagram 7: Command Routing

```mermaid
flowchart TD
  Command["Validated DialogueCommand"] --> Act{"act"}

  Act -->|answer| Answer["Must match current ExpectedAnswerSpec"]
  Act -->|change_step| Change["Target must be in active StepHistory, then enter target step"]
  Act -->|switch_intent| Switch["Target must be in active intent AllowedSwitchIntents, then delegate"]
  Act -->|unknown| Unknown["Reprompt / fallback"]

  Answer --> Handle["Current Step.Handle"]
  Handle --> StepResult["StepResult"]
  StepResult --> Apply["Orchestrator updates Lex SessionAttributes"]
```

## MakePayment Date Slots

| Payment type | Step | Slot | Canonical value |
| --- | --- | --- | --- |
| `onetime` | `getPaymentDate` | `paymentDate` | Full date, for example `2026-06-10` |
| `autopay` | `getAutopayDay` | `autopayDay` | Day of month, integer `1` through `31` |

Do not use one shared date slot for both branches. One-time payment and autopay have different disclosure wording, so they need different fields.

Also do not use one shared date step. `getPaymentDate` and `getAutopayDay` are two separate steps:

- `getPaymentDate.Handle` writes only `makePayment.paymentDate`.
- `getAutopayDay.Handle` writes only `makePayment.autopayDay`.

## Diagram 8: StepResult Apply Order

```mermaid
flowchart LR
  StepResult["StepResult"] --> Update["1. Update ConversationState\nset new values + clear stale values"]
  Update --> Transition["2. Apply transition\nactive step / delegate / resume"]
  Transition --> Persist["3. Write SessionAttributes"]
```

## Minimal Rules

| Topic | Rule |
| --- | --- |
| Entry intent | Demo runtime provides the first intent only when no active conversation exists. Future Lex can provide the same entry intent. |
| Runtime guard | If `ConversationState.activeIntent` exists, ignore conflicting incoming intent labels and keep routing through Orchestrator. |
| LLM | LLM only returns structured `DialogueCommand`. |
| Guard | `CommandGuard` validates command shape and canonical values against `ExpectedAnswerSpec`. |
| Yes/no | Yes/no is not a command act; it is an `answer` value only when the current step allows `yes` / `no`. |
| Step registry | Full `StepRegistry` is code-only; do not send all steps to the LLM. |
| Step history | LLM receives only already executed / returnable steps for the active intent. |
| Change step | `targetStep` must exist in both active intent `StepRegistry` and active intent `StepHistory`. |
| Switch intent | `targetIntent` must be listed in the active intent's allowed peer intent relationships. |
| Switch resume | During an active conversation, `switch_intent` defaults to `resumeBehavior=push_current_step`, so the original step resumes after the target intent completes. |
| Step | Step owns business logic and returns `StepResult`; it does not write Lex SessionAttributes. |
| Orchestrator | Only Orchestrator updates Lex SessionAttributes from `StepResult` and handles active step / delegate / resume. |
| Turn loop | One Lambda invocation loops until it has exactly one customer-facing message, completes, or transfers. |
| Loop guard | Stop the internal turn loop after `maxTurnIterations = 10` and use safe fallback / transfer. |
| Storage | `ConversationState` lives inside Lex `sessionAttributes`. |
| Phase | Persist only `entering_step` or `waiting_for_answer`; change/switch/resume always enters the target step as `entering_step`. |
| Retry | `retryCount` belongs to the current prompt. Increment on unknown/invalid/low confidence, max 3. Reset when step/intent/prompt identity changes; do not reset for same-prompt reprompt. |
| Resume | Resume only if `ResumeFrame` exists. |
| Resume target | Return to original intent + original step. |
| Resume execution | Rerun `Prepare -> Prompt`. |
| Change / switch | Current prompt expires; the target step enters with `phase=entering_step`. |

## Demo Scenarios

1. `MakePayment` happy path.
2. `MakePayment -> LinkExternalAccount -> resume MakePayment`.
3. Direct `LinkExternalAccount` completes without resuming `MakePayment`.
4. User changes amount/date/account after later steps; disclosure becomes stale.
5. User says `automatic pay`; LLM normalizes to `autopay`.
6. During disclosure, user says `sure`; LLM returns `act=answer`, `slot=disclosureAccepted`, `value=yes`.
7. During amount collection, user says `yes`; `CommandGuard` rejects it as `unknown`.
8. User changes step or switches intent; the old question expires and the target step runs `Prepare -> Prompt`.
9. A new incoming intent label conflicts mid-flow; `RuntimeIntentGuard` keeps the active intent from `ConversationState`.
10. LLM tries to jump to a step not in `StepHistory`; `CommandGuard` downgrades it to `unknown`.
