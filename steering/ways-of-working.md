# Ways of Working

## Rules of Engagement

- NEVER run any Docker command that deploys, updates, or mutates running services or stacks. This agent may run in production, where deployment-like operations are subject to manual approval.
- Prohibited (non-exhaustive): `docker stack deploy`, `docker service create`, `docker service update`, `docker service rm`, `docker stack rm`, `docker swarm init`/`join`/`leave`, `docker secret create`/`rm`, `docker config create`/`rm`, `docker network create`/`rm`, `docker volume rm`, `docker compose up`/`down`/`restart`, `docker run` that starts long-lived or privileged containers, and any `--force` service mutation.
- Allowed: read-only queries — `docker ps`, `docker service ls`/`ps`/`inspect`, `docker stack ls`/`ps`/`services`, `docker node ls`, `docker config/secret/volume/network ls`/`inspect`, `docker logs`, `docker info`, `docker compose config`. Transient read-only `docker run --rm` strictly for diagnostics (no ports, no privileges, no host mutation) is acceptable.
- Running `deploy.sh` (or any script that performs a deployment) is likewise prohibited — hand the command to the user to run under manual approval.
- When a change needs deploying to take effect, make the file changes, verify them statically (e.g. `docker compose config`), and tell the user the exact command to run. Do not run it yourself.

## During a Task

- Between receiving a task and presenting the result, produce NO prose except clarifying questions. All intermediate reasoning happens silently via tool use.
- Debugging is a task. Work silently through tool calls. Do not narrate hypotheses, intermediate findings, or what you're about to try.
- Conversational turns are not exceptions. Even if the user asks follow-up questions mid-task or the work spans multiple messages, the no-prose rule still applies to all tool-use turns.
- Never explain what you're about to do. Just do it.
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
