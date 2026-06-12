# Claude Code Prompt: Migrate Parser Prompt Assembly To ADK Instruction Templates

## Context

This task is for the intent / step orchestration demo described in:

- `docs/superpowers/specs/2026-06-07-intent-step-orchestration-design.md`
- `docs/superpowers/specs/2026-06-09-adk-tool-guard-requirements.md`
- `docs/superpowers/specs/2026-06-07-intent-step-orchestration-confluence.md`

Current state (do not re-implement, reuse):

- Go demo with `Orchestrator`, `Intent`/`Step` registry, `CommandGuard`, `RuntimeIntentGuard`, `ConversationState` carried in Lex `sessionAttributes`.
- `DialogueParser` wraps ADK Go (`google.golang.org/adk`, v1.4.x) with one `LlmAgent`, one shared `functiontool` named `emit_dialogue_command`, and Before/After Model/Tool callbacks.
- The per-turn LLM instruction is currently assembled by hand-building one big prompt string from: active_intent, active_step, ExpectedAnswerSpec, returnable StepHistory targets, AllowedSwitchIntents, canonical value mapping examples, and output rules (see the "Runtime Step Prompt Example" in the confluence doc).
- `session.InMemoryService()` is request-scoped per Parse call; customer conversation state stays in the orchestrator, NOT in ADK session.

## Goal

Replace the hand-concatenated prompt string with ADK Go's native instruction templating so the parser instruction is structured, testable, and assembled per step from state — without changing any architecture boundary.

Use the ADK Go instruction surface:

- `llmagent.Config.Instruction` static template with `{key}` and `{key?}` (optional) placeholders resolved from ADK session state.
- OR `llmagent.Config.InstructionProvider func(agent.ReadonlyContext) (string, error)` for dynamic assembly. NOTE: InstructionProvider output is NOT auto-resolved; the provider must call `instructionutil.InjectSessionState` explicitly.
- `util/instructionutil` package: `InjectSessionState(ctx, template)`.

Follow the installed ADK version's actual API names. The shapes above are contracts, not exact signatures to copy blindly.

## Required Design

### 1. ContextPack -> session state bridge

Before each Parse call, `DialogueParser` writes only safe, LLM-visible routing context into the request-scoped ADK session state.

Required scalar keys:

```text
active_intent
active_step
expected_slot
last_assistant_prompt_summary
```

Optional pre-rendered section keys:

```text
allowed_values_section
canonical_examples_section
returnable_steps_section
allowed_switch_intents_section
customer_context_section
```

Rules:

- Section keys must be fully rendered strings, including their own section header (for example `Allowed values:\n- onetime\n- autopay`).
- Empty optional sections must be omitted or set to empty string. Do not rely on `{key?}` to remove a header written elsewhere in the template.
- Keep rendering in one Go function per key so tests can assert exact output.
- Missing required key must fail fast (before any model call).
- Missing optional section key must not fail.
- Do not write `latest_user_utterance` into ADK session state. It must be passed only as the user message content for this invocation.
- Do not store the raw assistant question or disclosure text in ADK session state. Store only a safe summary in `last_assistant_prompt_summary`, for example: "The assistant was asking for payment type." / "The assistant was asking the user to accept the one-time payment disclosure." Raw disclosure/question text may contain amount/date/account values and would violate the no-PII rule.
- NEVER write into state: raw assistant disclosure/question text, full StepRegistry, unvisited steps, all registered intents, raw account numbers, customer identity, or IntentState payment amount/date values. Same exclusion list as the tool-guard doc "Prompt Context Requirement".

### 2. Instruction template

Create one static instruction template (Go `const` or embedded file) with placeholders instead of fmt.Sprintf:

```text
You are a dialogue command normalizer for a phone payment flow.
... (keep the existing role / not-the-flow-controller rules verbatim)

Current context:
- active_intent: {active_intent}
- active_step: {active_step}
- last_assistant_prompt_summary: {last_assistant_prompt_summary}

ExpectedAnswerSpec:
- slot: {expected_slot}

{allowed_values_section?}

{canonical_examples_section?}

{allowed_switch_intents_section?}

{returnable_steps_section?}

{customer_context_section?}

Do not infer or invent other steps. The full StepRegistry is code-only.

Output rules:
1. Call emit_dialogue_command exactly once.
2. Do not produce customer-facing text.
3. act must be one of: answer, change_step, switch_intent, unknown.
4. Do not use confirm or deny as acts.
5. If the user answered the current question, use act=answer.
6. For answer, slot must equal expected_slot.
7. For answer, value must be canonical and must be one of allowed_values when allowed_values is non-empty.
8. For change_step, target_step must come from returnable_steps only.
9. For switch_intent, target_intent must come from allowed_switch_intents only.
10. If the utterance cannot be mapped safely, emit unknown.
```

The latest user utterance stays in the user message, NOT in the instruction and NOT in session state.

### 3. Wiring

- Prefer `Instruction` static template + state injection if the installed version resolves `{key}`/`{key?}` from session state automatically.
- If any needed key cannot live in session state cleanly, use `InstructionProvider` + `instructionutil.InjectSessionState` and document why. Do not mix both resolution paths: if InstructionProvider is used, do not also rely on Instruction auto-injection.
- Keep all existing callbacks and the single `emit_dialogue_command` tool unchanged.
- Steps continue to produce `ExpectedAnswerSpec` + prompt context data; they must NOT produce final instruction text. The template owns wording; steps own data.

### 4. Constraints (unchanged invariants)

- LLM never mutates ConversationState, never writes sessionAttributes, never produces customer-facing text.
- `hooks.ValidateCommand` / Go `CommandGuard` remain the final gate.
- ADK session stays request-scoped; two Parse calls with different ContextPacks must not share state (existing Lambda-safety tests must still pass).
- No PII in logs: log template version + which optional sections were present (booleans), never the rendered values.

## Testing

Add/extend table-driven tests:

1. Renderer unit tests: each ContextPack key renders exactly as expected, including its own section header (empty input -> empty string -> section absent).
2. Template resolution test: given a fake ReadonlyContext/state, resolved instruction contains active step data and does NOT contain: full step registry names, unvisited step names, other intents not in AllowedSwitchIntents, raw disclosure text.
3. Optional-section test: step with no returnable steps and no switch intents -> those sections (including headers) absent from the resolved instruction.
4. Golden-file test: one fully resolved instruction for `MakePayment.choosePaymentType` matches a checked-in golden file (update the confluence example if wording changes).
5. Isolation test: two sequential Parse calls with different ContextPacks produce different resolved instructions and share no session state.
6. Regression: existing parser tests (normalization, switch_intent, unknown downgrade) still pass unchanged.
7. Required-key test: missing `active_step` or `expected_slot` fails before any model call.
8. Provider precedence test: if `InstructionProvider` is used, it calls `InjectSessionState` explicitly and the code does not also rely on `Instruction` auto-injection.

## Acceptance Criteria

- No ad-hoc full instruction prompt assembly remains in DialogueParser. Small section renderers may use strings.Join / formatting, but every renderer must be unit-tested.
- Steps supply data only; template supplies wording.
- LLM-visible context still matches the "LLM-Visible Routing Context" allowlist exactly; `last_assistant_prompt_summary` is a safe summary, never raw question/disclosure text.
- Required keys fail fast when missing; optional sections degrade silently.
- All existing tests pass; new tests above pass.
- Brief README note in the parser package: template version, key list (required vs optional), how to add a new key safely.

## Out Of Scope

- No read-only data tools, no LoopAgent/retry changes, no telemetry changes in this task.
- Do not change CommandGuard, Orchestrator, Step interfaces, or ConversationState schema.
