# UX swarm prompt

A prompt for Claude Code (or any coding agent) that builds a swarm of AI agents. The agents use your web app like real users and report where it breaks and where users have to stop and think.

**How to use it:** open your coding agent in your app's folder, fill in the `<...>` placeholders and paste the prompt. Leave a placeholder empty if you don't know the answer, and the agent will work it out from the repo and ask you.

**What you need:** a web app that runs locally or on staging, and an LLM API key. Every agent run costs tokens, so start with the small pilot round described in the prompt.

```text
I want to build a swarm of AI agents that use my web app like real users and find where it fails them, especially the moments where a user has to stop and think. This is not about unit tests: the agents must drive the real UI, the way a user would.

## Context
- App: <what it does, in one sentence>
- Who uses it: <e.g. small business owners, on desktop, several times a day>
- Stack: <e.g. Next.js + Postgres>
- How to run it locally: <command>, URL: <http://localhost:3000>
- Test data and test accounts: <seed script / sign-up flow / doesn't exist yet>
- Side effects to neutralize in tests: <emails, payments, SMS, webhooks, none>
- Real users' most frequent tasks, phrased the way they would say them:
  1. <task>
  2. <task>
  3. <task>

## Phase 0: explore and propose (stop and wait for my OK)
1. Read the repo and get the app running locally. Tell me if you can't.
2. Fill any gaps in the Context above from the code, and tell me what you assumed.
3. Propose the task list and personas (see below), and what you'll do about test data, test accounts and side effects.
4. Wait for my OK before building anything.

## Phase 1: build `ux-swarm/`
Plain scripts, no orchestration framework. One command runs a full round: `run → verify → aggregate`.

1. `tasks.yaml`: each task has an id, a goal written as the user would say it (never how to do it), a frequency (daily/weekly/monthly) and a step budget (how many steps a smooth run should take).
2. `personas.yaml`: start with these 5: a first-timer who skims and never reads help text; an impatient user who clicks the first plausible thing, sometimes twice; a messy-data user (accents, emojis, empty fields, very long text, pasted text with extra spaces); a user who quits halfway (reloads, hits back, opens a second tab) and then tries to continue; and a mobile user (375px viewport, touch).
3. `rules.md`: rules every agent gets.
   - Use the app only through its UI. Don't type URLs other than the start URL, don't read source code, don't use devtools.
   - At each step, write one line of inner monologue: what you're looking for, what you're about to click and how sure you are (high/medium/low).
   - These count as findings even when nothing breaks: hesitating between two options, a label that doesn't say what it does, an action with no visible confirmation, an error with no next step, losing what you typed, or going over the task's step budget.
   - Stay in character. Don't use knowledge a real user of this persona wouldn't have.
4. `run.py`: runner built on browser-use. Loops over personas × tasks in parallel with a configurable browser limit and `max_steps`. For each run, capture:
   - the agent's structured report (pydantic): completed, steps taken, and a list of findings with screen, step, expected, actual, type (broken / hesitation / no-feedback / lost-input / over-budget), severity, a quote from the inner monologue and a screenshot;
   - objective signals, collected automatically and not reported by the agent: JS console errors, failed network requests (4xx/5xx), back-navigations, repeated clicks on the same element, and steps vs. budget.
   Each run gets its own fresh test account and a known data state. Reset or reseed between runs so that agents can't affect each other.
   Severity: 3 = blocks the task, 2 = slows it down or causes an error the user recovers from, 1 = friction or cosmetic.
5. `verify.py`: re-runs every "broken" or "lost-input" finding with a fresh agent that only gets the reproduction steps. If it doesn't reproduce, mark it "doubtful". Subjective findings (hesitation, no-feedback) only count if 2 or more personas hit them, or if an objective signal backs them up.
6. `aggregate.py`: merges duplicates (same screen and same root cause) and writes `out/<date>/report.md` with:
   - a task × persona matrix: completed or failed, steps vs. budget;
   - the top 10 findings ranked by personas affected × task frequency × severity, each with screenshot, monologue quote, reproduction steps and a one-line suggested fix;
   - a comparison with the previous round, if there is one: new, fixed, still open;
   - a "doubtful" section at the end, kept apart from the rest.
7. `ignore.md`: findings I've reviewed and dismissed, one per line. `aggregate.py` filters them out so they don't come back every round.
8. `config.yaml`: models, parallelism, `max_steps`, start URL and a cost cap. Use a fast model for the agent runs and a strong one only for aggregation. Print the token usage and estimated cost at the end of each round, and stop if the cap is reached.
9. `README.md`: install, configuration, how to run a round, how to add tasks or personas, and how to dismiss findings.

## Constraints
- Agents run ONLY against local or staging, with test data. Never against production or real user data.
- Neutralize real side effects: emails to a local catcher, payments in test mode, external webhooks disabled or mocked. If you can't, tell me before running anything.
- Restrict agents to the app's own origin (a domain allowlist), so they can't wander onto external sites.
- API keys and test credentials go in environment variables, with a `.env.example` and the real `.env` in `.gitignore`.
- Before writing the runner, check the real API of the browser-use version you install by reading its code or docs. Don't rely on memory, because it changes often.
- Only use permissively licensed dependencies (see Notes). If you need another one, check its license and tell me before adding it. No GPL or AGPL.

## Phase 2: pilot round and honest review
- Run a pilot yourself: 3 personas × 3 tasks against the local app.
- Show me `report.md` and tell me honestly which findings look real and which are the agents' own mistakes, and why.
- If agents get stuck on the environment (login, captcha, missing data, rate limits), fix the test environment. Don't loosen the rules.
- Suggest how to run this every night or on every PR, but don't set it up until I say so.
- Don't commit until I say so.

## Notes: recommended stack (licenses checked on GitHub, 2026-11-24)
MIT:
- browser-use: LLM agent that drives the browser. The core of the swarm.
- pydantic: structured output schemas (Report, Finding).
- PyYAML: reading the YAML files.
- rich: progress and summary in the terminal (optional).
- typer: CLI for run/verify/aggregate (optional; stdlib argparse is enough).
- TypeScript alternative for the core: Stagehand (MIT), if the project is JS/TS.

Apache-2.0 (permissive, fine for commercial use, but not MIT):
- anthropic (Python SDK): only if you call Claude directly in aggregate.py. Not needed if you use the LLM client that ships with browser-use.
- Playwright: only if the installed browser-use version needs it, or for capturing console/network signals and seeding test data.
- uv: environment and dependency manager (optional; pip + venv works too).

Ruled out: Skyvern (AGPL-3.0), LaVague (abandoned), UXAgent (no license).
```

## Why it's built this way
- **Phase 0 stops before building**, so the agent doesn't build on wrong assumptions about your app.
- **Agents are smarter than real users.** The rules restrict them to the UI, keep them in character and make them log their confidence. That makes their hesitation something you can measure.
- **Agents' feelings are noisy, so hard signals back them up.** Console errors, failed requests, back-navigations and step budgets don't depend on what the agent says.
- **Verification and `ignore.md` cut the noise.** Reproduction filters out agent mistakes, and your dismissals stop repeat reports.
- **Real users stay in the loop.** The tasks come from them, and their priorities decide what gets fixed. The swarm only finds problems before they hit them.
