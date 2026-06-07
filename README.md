# Claude Code Prompt: Add Google ADK Go To Existing Model Integration

We already have a working Go demo that can call a company OpenAI-compatible model directly.

Current working behavior:

1. When the flow expects values such as "use checking" or "use card", if the user only says "checking", the current model parser can normalize it to the canonical value "checking".
2. During disclosure, if the user says they want to change the payment amount, the existing logic can go back, update the amount, invalidate disclosure, and continue the remaining flow such as collecting the payment date.

Do not rewrite the existing flow logic.
Do not rebuild the payment orchestrator.
Do not move payment business logic into the LLM.
Do not replace the existing CommandGuard, state machine, or downstream invalidation logic unless required for ADK integration.

Your task is to add Google ADK Go on top of the existing model connectivity and existing dialogue logic.

Target path:

```text
User input
-> ContextPack
-> ADK llmagent
-> OpenAIGatewayModel adapter
-> existing company OpenAI-compatible model client
-> structured command
-> existing Go guard / normalizer
-> existing Go orchestrator
-> next prompt
```

Required ADK integration:

1. Add an OpenAIGatewayModel adapter that implements google.golang.org/adk/model.LLM.
2. The adapter should reuse the existing company model connectivity logic:
   - same base URL configuration
   - same model name configuration
   - same app_id or application header behavior
   - same auth behavior
   - same certificate/truststore behavior if applicable
3. Create an ADK LLM agent using llmagent.New(...).
4. The ADK agent must use Model: OpenAIGatewayModel.
5. The ADK agent should receive the same ContextPack and latest user utterance that the current direct model parser receives.
6. The ADK agent must output the same structured command shape currently used by the existing flow.
7. Keep the existing Go validation layer after ADK:
   - canonical value validation
   - alias normalization
   - allowed slot validation
   - allowed command validation
   - current-step validation
   - downstream invalidation rules
8. Add a runtime switch:
   - direct mode: existing direct company model call
   - ADK mode: ADK llmagent using OpenAIGatewayModel

Required ADK callbacks:

Use ADK Go callbacks to make the ADK layer observable and safe. Do not use callbacks to replace the existing Go orchestrator.

Add these callbacks if supported by the installed ADK Go version:

1. BeforeAgentCallbacks

Purpose:

- Start an invocation trace.
- Log session id, active flow, active step, parser mode, and trace id.
- Do not log sensitive customer data.
- Do not mutate payment state.

2. BeforeModelCallbacks

Purpose:

- Confirm the model request is using the company gateway model.
- Ensure the ContextPack and latest user utterance are present.
- Optionally force deterministic model settings such as low temperature if the current ADK version supports it.
- Redact or avoid logging raw PII.
- Log request metadata, not full sensitive prompt content.

Expected behavior:

- The callback should not decide the next payment step.
- The callback should not execute business logic.
- The callback may reject or fail fast only if required model/context fields are missing.

3. AfterModelCallbacks

Purpose:

- Capture model response metadata.
- Log model version, latency, finish reason, token usage if available, and error status.
- Detect model failures or empty responses.
- Make it obvious in logs that the response came through the ADK model path.

Expected behavior:

- If the model response is empty or invalid, return or cause an "unknown" command path.
- Do not directly update payment state here.
- Do not treat AfterModel as the final business validator.

4. BeforeToolCallbacks

Purpose:

- Allow only the expected command-emitting tool, for example emit_dialogue_command.
- Validate the tool argument shape before accepting it.
- Log the proposed structured command.
- Reject unexpected tools.
- Optionally do early validation that command type, slot, and value exist.

Expected behavior:

- This is the best ADK callback location to inspect the structured command before it becomes parser output.
- Still keep the existing Go CommandGuard after ADK as the final validation layer.

5. AfterAgentCallbacks

Purpose:

- End the invocation trace.
- Confirm whether a structured command was emitted.
- Log total duration and parser outcome.
- If no command was emitted, make the parser return an unknown command.

Expected behavior:

- Use this for audit, logging, and fallback.
- Do not use this as the primary payment business validation point.

If the installed ADK Go version supports AfterToolCallbacks, add it only for logging tool results. If it does not support AfterToolCallbacks, do not invent one.

Acceptance criteria:

1. Existing direct company model connectivity still works.
2. New ADK path works and clearly uses:
   - google.golang.org/adk/agent/llmagent
   - google.golang.org/adk/model
   - OpenAIGatewayModel implements model.LLM
   - llmagent.New(... Model: gatewayModel ...)
3. ADK callback logs clearly show:
   - BeforeAgent invoked
   - BeforeModel invoked
   - AfterModel invoked
   - BeforeTool invoked when emit_dialogue_command is called
   - AfterAgent invoked
4. The "checking" case still works in ADK mode:
   - Current expected answer: use checking / use card
   - User says: checking
   - ADK path outputs structured command with canonical value: externalAccountType = checking
   - Existing Go guard accepts it and the flow moves to the next step
5. The disclosure correction case still works in ADK mode:
   - User is at disclosure
   - User says: I want to change the payment amount
   - ADK path outputs a structured command to update amount or go back to amount
   - Existing Go orchestrator updates amount, invalidates disclosure, and continues the flow by asking for the next required step such as payment date
6. No payment execution is added.
7. No real account creation is added.
8. No business policy decision is moved into ADK.
9. ADK is only used to parse and structure the user utterance through the company model.
10. The existing Go CommandGuard remains the final validator before state mutation.

Important design rule:

ADK should not become the payment flow controller.
ADK should be the structured command parser and observable model orchestration layer.
The existing Go orchestrator remains the source of truth for flow state and next step decisions.
