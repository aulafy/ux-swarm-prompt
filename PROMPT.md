# UX swarm prompt

Paste this into Claude Code (or any coding agent) from your app's folder. Fill in the `<...>` placeholders first.

```text
I want to set up a swarm of agents that use my app like real users and find where it fails from the user's point of view, especially the moments where the user has to think. This is not about unit tests: the agents must drive the real UI.

## Context
- App: <what it does, in one sentence>
- Stack: <e.g. Next.js + Postgres>
- How to run it locally: <command>, URL: <http://localhost:3000>
- How test data and test accounts are created: <seed script / sign-up / doesn't exist yet>
- I have one real user who relies on it daily to run her business. Her most frequent tasks are:
  1. <task, phrased the way she would say it>
  2. <task>
  3. <task>

## What I want you to build
A `ux-swarm/` folder in this repo with:

1. `tasks.yaml`: the tasks above, each with an id, a goal (without saying how to do it) and a frequency (daily/weekly/monthly).
2. `personas.yaml`: 5 personas: first-timer who doesn't read, impatient user who double-clicks, weird data (accents, emojis, empty fields, very long text), quits halfway (reloads or goes back, then tries to continue), and mobile (375px viewport).
3. `rules.md`: shared rules for every agent. These count as findings even if nothing breaks: hesitating between two options, an ambiguous label, an action with no visible confirmation, an error with no next step, or more than 3 steps for a frequent task.
4. `run.py`: runner using browser-use. Loops over personas × tasks in parallel with a configurable browser limit, and returns structured output via pydantic: completed, plus a list of findings with screen, step, expected, actual, type (broken/hesitation/no-feedback/too-many-steps), severity 1-3 and screenshot. Saves everything to `out/<date>/reports.json` along with the screenshots.
5. `verify.py`: re-runs every "broken" finding with a fresh agent that only gets the steps. If it doesn't reproduce, mark it "doubtful". "Hesitation" findings only count if they show up for 2 or more personas.
6. `aggregate.py`: groups duplicates by screen and ranks by personas affected × task frequency × severity. Writes `out/<date>/report.md` with the top 10, each with its screenshot and a one-line suggested fix.
7. `README.md`: how to install, set the API key via an environment variable, and run a round.

## Constraints
- Agents run ONLY against local or staging, with test data. Never against production or real data. If there's no way to seed test data, build one or ask me first.
- Before writing the runner, check the real API of the browser-use version you install (read its code or docs). Don't rely on memory, it changes often.
- Use a fast model for the agents in the runs and a strong one only for aggregation. Make the model configurable.
- The API key goes in an environment variable, never in the code.
- Keep it simple: plain scripts, no orchestration framework.
- Only use permissively licensed dependencies (see Notes). If you need another one, check its license and tell me before adding it. No GPL or AGPL.

## Definition of done
- Run a first small round yourself (3 personas × 3 tasks) against the app locally.
- Show me the resulting `report.md` and tell me honestly which findings look real and which are the agents' own mistakes.
- If agents get stuck on something environmental (login, captcha, missing data), fix it in the test environment instead of loosening the rules.
- Don't commit until I say so.

## Notes: recommended stack (licenses checked on GitHub, 2026-11-24)
MIT:
- browser-use: LLM agent that drives the browser. The core of the swarm.
- pydantic: structured output schemas (Report, Finding).
- PyYAML: reading tasks.yaml and personas.yaml.
- rich: progress and summary in the terminal (optional).
- typer: CLI for run/verify/aggregate (optional; stdlib argparse is enough).
- TypeScript alternative for the core: Stagehand (MIT), if the project is JS/TS.

Apache-2.0 (permissive, fine for commercial use, but not MIT):
- anthropic (Python SDK): only if you call Claude directly in aggregate.py. Not needed if you use the LLM client that ships with browser-use.
- Playwright: only if browser-use needs it in the installed version, or to seed test data via script.
- uv: environment and dependency manager (optional; pip + venv works too).

Ruled out: Skyvern (AGPL-3.0), LaVague (abandoned), UXAgent (no license).
```
