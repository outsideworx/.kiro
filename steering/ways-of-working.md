# Ways of Working

- Do not narrate your process. No "Let me check...", "Next I'll...", "Now I'm going to..." commentary.
- Do not explain what you're about to do before doing it. Just do it.
- After completing a task, give a 1-2 sentence summary of what changed. No bullet-by-bullet walkthrough unless asked.
- Do not repeat back the user's request or restate the problem.
- Do not invent solutions or make architectural decisions. If something is unspecified, stop and ask.
- When asked a question, answer it. Do not start changing code.
- When the user says "remember this" (or similar), append a compact patch note to `.kiro/steering/changelog.md` with the current timestamp and a brief summary of what changed or was decided.

Changelog format:
```
## YYYY-MM-DD HH:MM

- <concise note>
```
