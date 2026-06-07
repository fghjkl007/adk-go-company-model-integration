# Response To Claude Code: Section 1 Architecture

Proceed, but adjust Section 1 before sketching components.

Keep the overall direction: one new parser file, one demo-cli wiring edit, and reuse the existing OpenAIGatewayModel adapter if it already implements the ADK model.LLM contract. The important correction is that the ADK parser must be Lambda-safe and request-scoped.

Please update Section 1 with these rules:

1. Keep the CommandParser interface unchanged.
2. Keep openai_parser.go unchanged as the direct JSON-schema baseline.
3. Keep normalizing_parser.go in the tree, but it should no longer be the default demo path.
4. Add internal/parser/adk_parser.go as the ADK-backed CommandParser.
5. Reuse internal/adkbridge/openai_llm.go only if it is stateless and already satisfies google.golang.org/adk/model.LLM.
6. cmd/demo-cli/main.go should use PARSER_MODE=direct|adk.
7. PARSER_MODE=direct should wire NewOpenAIParser, not NewNormalizingParser.
8. PARSER_MODE=adk should wire NewADKParser.
9. Unknown PARSER_MODE values should fail fast with a clear error.

For Lambda safety, be explicit:

- Do not store per-customer state in package globals.
- Do not store current ContextPack, user utterance, captured command, tool result, ADK session, payment draft, active step, or Lex session data in shared parser fields.
- A warm Lambda container may serve different customers across invocations, so anything customer-specific must be rebuilt or loaded per request.
- The only shared objects allowed across Parse calls are stateless infrastructure objects, such as the OpenAIGatewayModel adapter, HTTP client, immutable config, logger, and metrics client.
- The ADK session for the parser should be one-shot per Parse call and discarded after that call.
- The durable conversation/payment state remains outside ADK, loaded from the existing Lex/session persistence path and passed into the parser as ContextPack.

For internal/parser/adk_parser.go, use Approach A:

- NewADKParser(cfg ADKConfig) returns a CommandParser.
- ADKConfig should mirror the existing OpenAI parser config names as much as possible: API key, base URL, app ID, model, and any required gateway headers.
- The parser can hold the shared stateless model adapter and rendered instruction template.
- Each Parse(ctx, input) call must create a fresh local capturedCommand variable.
- Each Parse(ctx, input) call must create a fresh emit_dialogue_command tool whose handler writes only to that local capturedCommand variable.
- Each Parse(ctx, input) call should build or run a one-shot llmagent with that request-local tool.
- The ADK agent receives the same ContextPack JSON and latest user utterance that openai_parser.go sends today.
- After the agent run finishes, return capturedCommand if the tool fired.
- If the tool did not fire, return Unknown.
- Always pass the result through the existing Go guard / hooks.ValidateCommand path before the orchestrator mutates state.

For callbacks:

- Add BeforeAgentCallbacks for trace start and redacted request metadata.
- Add BeforeModelCallbacks to verify model/context presence and log sanitized model-request metadata.
- Add AfterModelCallbacks for latency, model response status, token usage if available, and empty/invalid model response detection.
- Add BeforeToolCallbacks to allow only emit_dialogue_command and perform early shape validation of the proposed command.
- Add AfterAgentCallbacks to close the trace and detect whether a command was emitted.
- Add AfterToolCallbacks only if the installed ADK Go version supports it. If not supported, do not invent it.

Callback boundaries:

- Callbacks are for observability, audit, and early safety checks.
- Callbacks must not decide the payment next step.
- Callbacks must not mutate payment state.
- Callbacks must not replace CommandGuard.
- The existing Go orchestrator remains the only source of truth for flow movement, step rollback, switch intent behavior, and downstream invalidation.

Please continue with the component sketch after making those adjustments.

The component sketch should show:

1. demo-cli parser selection with PARSER_MODE=direct|adk.
2. ADKParser implementing CommandParser.
3. OpenAIGatewayModel implementing model.LLM.
4. llmagent.New using the gateway model, command-emitting tool, and callbacks.
5. emit_dialogue_command tool returning the existing dialogue.Command shape.
6. Existing hooks.ValidateCommand / CommandGuard after ADK.
7. Existing Go orchestrator applying the validated command.
8. No business flow logic inside ADK.

Acceptance checks to preserve:

1. Direct mode still works.
2. ADK mode still normalizes "checking" into the existing canonical command value.
3. ADK mode still handles disclosure correction by emitting a command that the existing Go orchestrator can use to go back to amount, invalidate disclosure, and continue.
4. No customer-specific state leaks across Parse calls or Lambda invocations.
5. Logs prove the ADK callbacks ran without exposing sensitive customer data.
