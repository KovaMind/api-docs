# Create a ChatGPT Custom GPT with Kova Mind Memory

Give any Custom GPT persistent memory across conversations using Kova Mind.

## Setup (5 minutes)

### 1. Get your API key

Sign up at [kovamind.ai](https://kovamind.ai) and copy your key from **Settings > API Keys**.

### 2. Create a new GPT

Go to [ChatGPT GPT Editor](https://chatgpt.com/gpts/editor) and click **Create**.

### 3. Add the Action

In the GPT editor, click **Configure** > **Create new action**.

Paste this OpenAPI schema URL:

```
https://kovamind.github.io/api-docs/openapi.yaml
```

Or paste the schema directly — copy the contents of [`openapi.yaml`](./openapi.yaml).

### 4. Add authentication

In the Action settings:
- **Authentication type**: API Key
- **Auth Type**: Bearer
- **API Key**: Your `km_live_xxx` key

### 5. Set the GPT instructions

Paste this into your GPT's **Instructions** field:

```
You have access to a persistent memory system via the Kova Mind API.

MEMORY RULES:
1. At the START of every conversation, call retrieveMemory with the user's first message as context and user_id "{{user_id}}" to recall relevant memories.
2. After each user message, decide if new information was shared. If yes, call extractMemory with the conversation so far.
3. If the user confirms or denies something you remembered, call reinforceMemory on that pattern.
4. If the user says something that contradicts your memory, call scoreSurprise first to evaluate, then extract the new information.

BEHAVIOR:
- Never mention "memory API" or "patterns" to the user. Just naturally remember things.
- When recalling, weave memories naturally into your responses.
- User ID: use "{{user_id}}" for all API calls.
```

Replace `{{user_id}}` with a unique identifier (their email, username, or a UUID).

### 6. Test it

Start a conversation:

> **User**: I'm a Python developer who works at a startup in Austin. I prefer tabs over spaces.
>
> **GPT**: *(calls extractMemory, stores 3 patterns)*
> Great to meet you! What kind of work does your startup do?

Start a **new** conversation:

> **User**: Can you recommend a code editor for me?
>
> **GPT**: *(calls retrieveMemory, recalls: Python developer, prefers tabs)*
> Since you're working with Python and prefer tabs, I'd suggest...

The GPT remembers across conversations without the user doing anything.

## Advanced: Multi-user GPTs

If your GPT serves multiple users, generate a unique `user_id` per user. Options:

- Use their email: `user_id: "alex@example.com"`
- Generate a UUID on first visit and store it in the conversation
- Use an external auth system's user ID

## Limitations

- ChatGPT Custom GPT Actions have a ~25 second timeout per API call
- The GPT can only call actions when it decides to — it may not always remember to recall
- Free tier: 100 extractions/day, 200 retrievals/day
