# MCU Agentic Workflow

An OpenCode agent setup that runs engineering work like a small production team:
a **Director** that only delegates, **Avengers** that plan and investigate,
**Defender** that implements, and **Shield** that gates quality.

The primary agent never writes code and the worker agents never talk to the user
directly. Work only moves forward through explicit roles and an approval gate,
which makes AI-assisted changes easier to audit and less likely to quietly drift.

## Roles

| Agent | Mode | Model | Job |
| --- | --- | --- | --- |
| `director` | primary | `deepseek/deepseek-v4-pro#high` | Classifies work, delegates, relays results. Cannot read, edit, or run shell. |
| `avengers/kimi` | subagent | `kimi-code-plan-global/kimi-for-coding#high` | Plans, architecture, debugging analysis, research, exploration (slot 1). |
| `avengers/terra` | subagent | `openai/gpt-5.6-terra#xhigh` | Same as above (slot 2 fallback). |
| `defender/kimi` | subagent | `kimi-code-plan-global/kimi-for-coding#high` | Implements approved plans, writes unit tests (slot 1). |
| `defender/flash` | subagent | `deepseek/deepseek-flash#high` | Same as above (slot 2 fallback). |
| `shield/kimi` | subagent | `kimi-code-plan-global/kimi-for-coding#high` | QA review and quality gate (slot 1). |
| `shield/sol` | subagent | `openai/gpt-5.6-sol#high` | Same as above (slot 2 fallback). |

Avengers and Shield are read-only: they may read anything and run tests, but
they cannot edit or write files.

## How the workflow runs

1. **Classify** — the Director decides whether the request is implementation,
   research, or planning-only.
2. **Plan** — for implementation work, Avengers produce a concrete plan (files,
   approach, test strategy). The Director relays it and stops for human approval.
   Unapproved plans never reach Defender.
3. **Implement** — Defender executes the approved plan and runs the relevant tests.
4. **Review** — Shield returns `PASS` or `FAIL` against the original request.
   On `FAIL`, findings go back to Defender and Shield reviews again, for up to
   three rounds.
5. **Research/analysis** — routed straight to Avengers, no approval gate.

Model fallback happens only when a dispatch itself fails (provider error, rate
limit, unreachable model, timeout). A completed but disappointing answer is never
re-delegated to another model.

## Repository layout

```
.
├── opencode.jsonc        # superpowers plugin
├── cli.json              # terminal UI preferences (theme, diffs, sidebar)
└── agents/
    ├── director.md
    ├── avengers/{kimi,terra}.md
    ├── defender/{kimi,flash}.md
    └── shield/{kimi,sol}.md
```

## Install

Requires OpenCode v2.x. Back up any existing global config first, then clone this
repository into the global config directory:

```sh
mv ~/.config/opencode ~/.config/opencode.bak 2>/dev/null
git clone git@github2:loidforger13/opencode-mcu-agentic.git ~/.config/opencode
```

Using HTTPS instead of SSH:

```sh
git clone https://github.com/loidforger13/opencode-mcu-agentic.git ~/.config/opencode
```

Then authenticate the providers, which are per machine and never committed:

```sh
opencode auth login          # Kimi For Coding and DeepSeek
opencode auth login          # OpenAI, if you want the terra/sol fallbacks
```

Verify the plugin loaded and start a session with the Director:

```sh
opencode plugin list         # should list superpowers
```

```sh
opencode
```

Select `director` as the agent. The `superpowers` plugin installs itself from its
git URL on first load.

## Notes

- `service.json` is machine-local and contains a credential — it is gitignored
  and must never be committed.
- The committed config uses the `"plugin"` key because that is what OpenCode
  `v2.2.6` reads; `"plugins"` is ignored by that build.
- `avengers/*` and `shield/*` deny `edit`/`write`/`subagent` in their frontmatter.
  Keep those permissions if you edit the roles.
- Fallback models only matter if you use those providers. Without OpenAI auth,
  `avengers/terra` and `shield/sol` simply will not dispatch; the slot-1 models
  still work.

## Customizing

Role behavior lives in the agent Markdown bodies, model choices in each file's
`model:` frontmatter, and delegation rules in `agents/director.md`. Config
changes are picked up without restarting OpenCode.
