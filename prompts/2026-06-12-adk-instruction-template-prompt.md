# Claude Code Prompt: Migrate Parser Prompt Assembly To ADK Instruction Templates

## Context

This task is for the intent / step orchestration demo described in:

- `docs/superpowers/specs/2026-06-07-intent-step-orchestration-design.md`
- `docs/superpowers/specs/2026-06-09-adk-tool-guard-requirements.md`
- `docs/superpowers/specs/2026-06-07-intent-step-orchestration-confluence.md`

Current state (do not re-implement, reuse):

- Go demo with `Orchestrator`, `Intent`/`Step` registry, `CommandGuard`, `RuntimeIntentGuard`, `ConversationState` carried in Lex `sessionAttributes`.
- `DialogueParser` wraps ADK Go (`google.golang.org/adk`, v1.4.x) with one `LlmAgent`, one shared `functiontool` named `emit_dialogue_command`, and Before/After Model/Tool callbacks.
- The per-turn LLM instruction is currently assembled by hand-building one big prompt string from: active_intent, active_step, last_assistant_question, ExpectedAnswerSpec, returnable StepHistory targets, AllowedSwitchIntents, canonical value mapping examples, and output rules (see the "Runtime Step Prompt Example" in the confluence doc).
- `session.InMemoryService()` is request-scoped per Parse call; customer conversation state stays in the orchestrator, NOT in ADK session.

## Goal

Replace the hand-concatenated prompt string with ADK Go's native instruction templating so the parser instruction is structured, testable, and assembled per step from state — without changing any architecture boundary.

Use the ADK Go instruction surface:

- `llmagent.Config.Instruction` static template with `{key}` and `{key?}` (optional) placeholders resolved from ADK session state.
- OR `llmagent.Config.InstructionProvider func(agent.ReadonlyContext) (string, error)` for dynamic assembly. NOTE: InstructionProvider output is NOT auto-resolved; call `instructionutil.InjectSessionState` manually if the provider returns a template with placeholders.
- `util/instructionutil` package: `InjectSessionState(ctx, template)`.

Follow the installed ADK version's actual API names. The shapes above are contracts, not exact signatures to copy blindly.

## Required Design

### 1. ContextPack -> session state bridge

Before each Parse call, `DialogueParser` writes the turn's safe routing context into the request-scoped ADK session state (the same `InMemoryService` session it already creates per call):

```text
active_intent
active_step
last_assistant_question
expected_slot
allowed_values            (rendered list, may be empty)
canonical_examples        (rendered mapping lines for the current step)
returnable_steps          (rendered list with one-line descriptions; active intent StepHistory only)
allowed_switch_intents    (rendered list with one-line descriptions; active intent Relationships only)
customer_context_summary  (safe summary only)
```

Rules:

- Values must be pre-rendered strings (lists joined with newlines/bullets). Keep rendering in one Go function per key so tests can assert exact output.
- Optional sections use `{key?}` so an empty key disappears instead of rendering an empty header.
- NEVER write into state: full StepRegistry, unvisited steps, all registered intents, raw account numbers, customer identity, amounts/dates from IntentState. Same exclusion list as the tool-guard doc "Prompt Context Requirement".

### 2. Instruction template

Create one static instruction template (Go `const` or embedded file) that mirrors the existing "Runtime Step Prompt Example" structure in the confluence doc, with placeholders instead of fmt.Sprintf:

```text
You are a dialogue command normalizer for a phone payment flow.
... (keep the existing role / not-the-flow-controller rules verbatim)

Current context:
- active_intent: {active_intent}
- active_step: {active_step}
- last_assistant_question: "{last_assistant_question}"

ExpectedAnswerSpec:
- slot: {expected_slot}
{allowed_values?}

{canonical_examples?}

{allowed_switch_intents?}

{returnable_steps?}

Do not infer or invent other steps. The full StepRegistry is code-only.

Output contract:
... (keep the existing 10 output rules verbatim, including yes/no-as-answer rule)
```

The latest user utterance stays in the user message, NOT in the instruction.

### 3. Wiring

- Prefer `Instruction` static template + state injection if the installed version resolves `{key}`/`{key?}` from session state automatically.
- If any needed key cannot live in session state cleanly, use `InstructionProvider` + `instructionutil.InjectSessionState` and document why.
- Keep all existing callbacks and the single `emit_dialogue_command` tool unchanged.
- Steps continue to produce `ExpectedAnswerSpec` + prompt context data; they must NOT produce final instruction text. The template owns wording; steps own data.

### 4. Constraints (unchanged invariants)

- LLM never mutates ConversationState, never writes sessionAttributes, never produces customer-facing text.
- `hooks.ValidateCommand` / Go `CommandGuard` remain the final gate.
- ADK session stays request-scoped; two Parse calls with different ContextPacks must not share state (existing Lambda-safety tests must still pass).
- No PII in logs: log template version + which optional sections were present (booleans), never the rendered values.

## Testing

Add/extend table-driven tests:

1. Renderer unit tests: each ContextPack key renders exactly as expected (empty list -> empty string -> section omitted).
2. Template resolution test: given a fake ReadonlyContext/state, resolved instruction contains active step data and does NOT contain: full step registry names, unvisited step names, other intents not in AllowedSwitchIntents.
3. Optional-section test: step with no returnable steps and no switch intents -> those sections absent from the resolved instruction.
4. Golden-file test: one fully resolved instruction for `MakePayment.choosePaymentType` matches a checked-in golden file (update the confluence example if wording changes).
5. Isolation test: two sequential Parse calls with different ContextPacks produce different resolved instructions and share no session state.
6. Regression: existing parser tests (normalization, switch_intent, unknown downgrade) still pass unchanged.

## Acceptance Criteria

- No fmt.Sprintf / string-concat prompt assembly remains in DialogueParser; instruction comes from one template + state injection.
- Steps supply data only; template supplies wording.
- LLM-visible context still matches the "LLM-Visible Routing Context" allowlist exactly.
- All existing tests pass; new tests above pass.
- Brief README note in the parser package: template version, key list, how to add a new key safely.

## Out Of Scope

- No read-only data tools, no LoopAgent/retry changes, no telemetry changes in this task.
- Do not change CommandGuard, Orchestrator, Step interfaces, or ConversationState schema.
