# Intent / Step Orchestration Demo

## Purpose

Small demo design for showing how a Go Lambda can manage multi-step intents after Lex has already classified the first intent.

Key idea:

```text
Lex classifies the entry intent.
LLM normalizes the user's answer into a DialogueCommand.
Go Orchestrator reads Lex SessionAttributes, validates/routes, updates ConversationState, and writes updated SessionAttributes.
```

## Demo Scope

- Entry intent comes from Lex.
- Demo intents: `MakePayment` and `LinkExternalAccount`.
- Intents are same-level; one intent does not own another.
- `MakePayment` can route to `LinkExternalAccount`.
- Resume happens only when a `ResumeFrame` exists.
- On resume, return to the original intent and original step, then rerun `Prepare -> Prompt`.
- Demo `ConversationState` is stored in Lex `sessionAttributes`.

## Diagram 1: Runtime Architecture

```mermaid
flowchart TD
  User["User utterance"] --> Lex["Lex classifies entry intent"]
  Lex --> Lambda["Go Lambda receives Lex intent + utterance + session id"]
  Lambda --> SessionRead["Read Lex SessionAttributes"]
  SessionRead --> Orchestrator["Orchestrator"]
  Orchestrator --> Active["Resolve active intent + active step"]
  Active --> Spec["Current Step + ExpectedAnswerSpec"]
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

## Diagram 2: Step Lifecycle

```mermaid
flowchart TD
  Enter["Enter step"] --> Prepare["Prepare: API reads / business checks"]
  Prepare --> Prompt["Prompt: message + ExpectedAnswerSpec"]
  Prompt --> Wait["Wait for user input"]
  Wait --> Parse["ADK / LLM returns normalized DialogueCommand"]
  Parse --> Guard["CommandGuard"]
  Guard --> Act{"Command act"}
  Act -->|answer / confirm / deny| Handle["Step.Handle"]
  Act -->|change_step / switch_intent| Route["Orchestrator routing"]
  Act -->|unknown| Reprompt["Reprompt / fallback"]
  Handle --> Result["StepResult"]
  Result --> Apply["Orchestrator writes updated SessionAttributes"]
  Route --> Enter
  Reprompt --> Wait
```

## Diagram 3: MakePayment To LinkExternalAccount

```mermaid
sequenceDiagram
  participant User
  participant Lex
  participant Orchestrator
  participant MakePayment
  participant LinkExternalAccount
  participant SessionAttributes as Lex SessionAttributes

  User->>Lex: "I want to make a payment"
  Lex->>Orchestrator: Lex intent = MakePayment
  Orchestrator->>MakePayment: Start / continue MakePayment
  MakePayment->>User: "Use your previous account or link a new account?"
  User->>Orchestrator: "link a new account"
  Orchestrator->>SessionAttributes: Write ResumeFrame(makePayment, currentStep)
  Orchestrator->>LinkExternalAccount: Start LinkExternalAccount
  LinkExternalAccount->>Orchestrator: CompleteIntent + StepResult
  Orchestrator->>SessionAttributes: Apply StepResult + pop ResumeFrame
  Orchestrator->>MakePayment: Resume original step
  MakePayment->>MakePayment: Rerun Prepare -> Prompt
  Orchestrator->>User: Ask refreshed MakePayment question
```

## Diagram 4: Direct LinkExternalAccount

```mermaid
sequenceDiagram
  participant User
  participant Lex
  participant Orchestrator
  participant LinkExternalAccount
  participant SessionAttributes as Lex SessionAttributes

  User->>Lex: "I want to link an account"
  Lex->>Orchestrator: Lex intent = LinkExternalAccount
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
  Conv --> Stack["SuspensionStack"]

  Active --> ActiveEx["Example: activeIntent=makePayment, activeStep=getPaymentDate"]

  Customer --> CustomerEx["Example: hasExternalAccount=true, balance=120.50"]

  Intents --> MP["MakePayment IntentState"]
  MP --> MPEx["Example: paymentType=onetime, amount=20, date=tomorrow"]

  Intents --> LEA["LinkExternalAccount IntentState"]
  LEA --> LEAEx["Example: accountType=checking, verificationStatus=verified"]

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
  Conv --> Stack["SuspensionStack"]

  Active --> ActiveValue["activeIntent=linkExternalAccount, activeStep=collectAccountType"]
  Intents --> MP["makePayment: paymentType=onetime, amount=20, paymentAccountRef=previousAccount"]
  Intents --> LEA["linkExternalAccount: accountType=pending, verificationStatus=pending"]
  Customer --> Ctx["externalAccounts=[previousAccount], balance=120.50"]
  Stack --> Frame["ResumeFrame: makePayment.choosePaymentAccount"]
  Frame --> Resume["On completion: resume makePayment.choosePaymentAccount and rerun Prepare -> Prompt"]
```

## Diagram 7: Command Routing

```mermaid
flowchart TD
  Command["Validated DialogueCommand"] --> Act{"act"}

  Act -->|answer| Answer["Must match current ExpectedAnswerSpec"]
  Act -->|confirm / deny| Confirm["Step must allow confirm / deny"]
  Act -->|change_step| Change["Move to target step + invalidate downstream data"]
  Act -->|switch_intent| Switch["Validate relationship + activate target intent"]
  Act -->|unknown| Unknown["Reprompt / fallback"]

  Answer --> Handle["Current Step.Handle"]
  Confirm --> Handle
  Handle --> StepResult["StepResult"]
  StepResult --> Apply["Orchestrator updates Lex SessionAttributes"]
```

## Diagram 8: StepResult Apply Order

```mermaid
flowchart LR
  StepResult["StepResult"] --> Validate["1. Validate"]
  Validate --> Data["2. Apply data"]
  Data --> Invalidate["3. Invalidate stale fields"]
  Invalidate --> Transition["4. Apply transition"]
  Transition --> Persist["5. Write SessionAttributes"]
```

## Minimal Rules

| Topic | Rule |
| --- | --- |
| Entry intent | Lex provides the first intent. |
| LLM | LLM only returns structured `DialogueCommand`. |
| Guard | `CommandGuard` validates before updating Lex SessionAttributes. |
| Step | Step returns `StepResult`; it does not write Lex SessionAttributes. |
| Orchestrator | Only Orchestrator updates Lex SessionAttributes from `StepResult`. |
| State storage | `ConversationState` lives inside Lex `sessionAttributes`. |
| Resume | Resume only if `ResumeFrame` exists. |
| Resume target | Return to original intent + original step. |
| Resume execution | Rerun `Prepare -> Prompt`. |

## Demo Scenarios

1. `MakePayment` happy path.
2. `MakePayment -> LinkExternalAccount -> resume MakePayment`.
3. Direct `LinkExternalAccount` completes without resuming `MakePayment`.
4. User changes amount/date/account after later steps; disclosure becomes stale.
5. User says `automatic pay`; LLM normalizes to `autopay`.
