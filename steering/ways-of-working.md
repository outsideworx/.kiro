# Ways of Working

## During a Task

- Between receiving a task and presenting the result, produce NO prose except clarifying questions. All intermediate reasoning happens silently via tool use.
- For multi-repo changes, proceed without confirmation unless the approach is ambiguous.
- If something is unspecified, stop and ask. Do not invent solutions or make architectural decisions.
- Clarifying questions are short and direct — no analysis, pros/cons, or numbered options unless asked.

## After a Task

- Give a 1-2 sentence summary of what changed. No headers, no diagrams, no code blocks, no bullet-by-bullet walkthrough unless asked.
- Do not repeat back the user's request or restate the problem.

## When Asked a Question

- Answer it. Do not start changing code.
- Keep answers concise and proportional to the question. Open-ended questions get a focused list, not an essay.

## Changelog

- When the user says "remember this" (or similar), append a compact patch note to `.kiro/steering/changelog.md` with the current timestamp and a brief summary of what changed or was decided.
- Do not write to the changelog automatically after completing tasks. It is the user's notebook, not an activity log.

Format:
```
## YYYY-MM-DD HH:MM

- <concise note>
```
