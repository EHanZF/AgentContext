# Agent Skills

## Gemini API

**Description:** This skill allows the agent to interact with the Gemini API to perform various tasks, such as generating text, translating languages, and answering questions.

**MCP Tool:** `gemini_api`

**Inputs:**
- `prompt`: The prompt to send to the Gemini API.
- `model`: The Gemini model to use (e.g., "gemini-pro").
- `max_tokens`: The maximum number of tokens to generate.

**Outputs:**
- `text`: The generated text from the Gemini API.
- `usage`: Token usage information.
