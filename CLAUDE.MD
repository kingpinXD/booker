## Related Repositories

- **Anuma AI Portal** (backend): `/Users/tanmay/IdeaProjects/jedi2002/ai-portal`
  - Go backend for the Anuma LLM router service
  - OpenAI-compatible API at `POST /api/v1/chat/completions`
  - Auth via `X-API-Key: anuma_live_<key_id>.<secret>` header
  - Supports models: `auto`, `gpt-4o-mini`, `claude-3-5-sonnet`, `gpt-4o`, `o1`
  - Routes through Bifrost multi-provider gateway (OpenAI, Anthropic, Google, Mistral)

## Code Standards

### General Principles
- Follow Go best practices and idiomatic patterns
- Write concise, readable, and maintainable code
- All exported types/functions must have comments
- Package comments required at top of files
- Follow TEST DRIVEN DEVELOPMENT when adding new features clarify the expected outcome always if you are not clear.
- Always use TODO's when working on adding new code 

**Important**: Only write code explicitly asked for during the prompt. Do not take initiatives to modify or create code not requested during sessions.

### Modifying Existing Code
- **Always modify existing functions** instead of creating new duplicate functions with slightly different signatures
- When adding a new parameter to a function, update the existing function signature rather than creating a `FunctionWithNewParam` variant
- If a function needs an optional parameter, add it with a sensible default or use empty/zero values to indicate "not provided"
- Avoid code duplication - if two functions share 90% of their logic, consolidate them into one.
- Prefer switch cases over nested if else statements
- Prefer early returns


**Important** when adding any code run an analysis to see if the number of lines of code added can be reduced in any way,
whether it is extracting common logic into functions, simplifying logic
**Important** prioritize simple readable code, overcomplicated ones.
It is okay to not handle cases which are unrealistic in favor of keeping the logic simple 

