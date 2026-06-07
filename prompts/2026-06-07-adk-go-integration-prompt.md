# Response To Claude Code: Section 1 Architecture

Proceed, but adjust Section 1 with these key points before the component sketch:

1. Lambda safety

- The ADK parser must be request-scoped.
- Do not keep ContextPack, user utterance, captured command, ADK session, payment state, active step, or Lex session data in globals or shared parser fields.
- Warm Lambda containers can be reused for different customers.
- Only stateless objects can be shared: model adapter, HTTP client, immutable config, logger, metrics client.
- Create a fresh capturedCommand, emit_dialogue_command tool, and one-shot ADK run per Parse call.

2. Parser wiring

- Keep CommandParser unchanged.
- Keep openai_parser.go as the direct JSON-schema baseline.
- PARSER_MODE=direct should use NewOpenAIParser.
- PARSER_MODE=adk should use NewADKParser.
- normalizing_parser.go can stay in the repo, but should not be the default demo path.

3. ADK responsibility

- ADK should only parse the utterance into the existing dialogue.Command shape.
- ADK should not control the payment flow.
- Existing Go CommandGuard / hooks.ValidateCommand remains the final validator.
- Existing Go orchestrator still owns next step, rollback, switch intent, and downstream invalidation.

4. Callbacks

- Add BeforeAgent, BeforeModel, AfterModel, BeforeTool, and AfterAgent callbacks.
- Use them for trace logging, sanitized metadata, model status, tool allowlist, and early command-shape checks.
- Do not mutate payment state inside callbacks.
- Add AfterTool only if the installed ADK Go version supports it.

5. Acceptance checks

- direct mode still works.
- adk mode still normalizes "checking" to the existing canonical command value.
- adk mode still handles disclosure correction by emitting a command the existing orchestrator can use to go back to amount and continue.
- No customer-specific state leaks across Parse calls or Lambda invocations.

Then continue with the component sketch.
