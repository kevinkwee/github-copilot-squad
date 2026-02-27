# GitHub Copilot Squad

A custom multi-agent orchestration setup for **GitHub Copilot coding agent**.

This repo defines a 4-agent team with clear roles:
- **Capybara**: router/orchestrator
- **Otter**: implementation agent
- **Owl**: reviewer/QA gate
- **Squirrel**: general assistant for simple/non-technical requests

The goal is to separate execution and review, so technical tasks go through an implementation + review loop before final output, with different models contributing different perspectives.

## Why this exists

When using one agent for everything, prompts often mix planning, coding, and reviewing in one pass, and the context window can fill up quickly. This setup enforces role separation:
- Technical work is done by a builder agent (`Otter`)
- Quality is validated by a strict reviewer (`Owl`)
- Implementation and review can run on different models, giving a multi-perspective quality check
- Non-technical/simple requests are handled fast by a lightweight helper (`Squirrel`)
- Routing and iteration control is centralized in `Capybara`

## Repository contents

- `Capybara.agent.md` — orchestration/router profile
- `Otter.agent.md` — coding/implementation profile
- `Owl.agent.md` — review/QA profile
- `Squirrel.agent.md` — general help profile

## Quick start

### Prerequisites
- GitHub Copilot Chat extension with custom agents support
- VS Code

### Option A — Repo-level agents (recommended)
Store the agent profiles in:
- `.github/agents/*.agent.md`

This makes them available for that repository/workspace.

### Option B — User-level agents
Create/store agent profiles in your user data custom agents location from **Configure Custom Agents...** button in VS Code.

This makes them available across your workspaces.

### Use it
1. Open Copilot Chat in VS Code.
2. Select `Capybara` from the agents dropdown.
3. Ask your request naturally.
4. For technical requests, `Capybara` orchestrates Otter ↔ Owl iterations automatically.

## How the flow works

### 1) Routing
`Capybara` decides where to send the request:
- **Technical implementation request** → `Otter`
- **Simple/non-technical request** → `Squirrel`
- **Ambiguous request** → asks a clarification question first

### 2) Technical orchestration loop (Otter ↔ Owl)
For technical requests, `Capybara` runs this loop:
1. Create report folder path: `.github/temp_reports/{YYYYMMDD_HHmmss}_{objective}/`
2. Send task to `Otter` with `{iteration}` and report path
3. Send results to `Owl` for review
4. If `APPROVED` → return final result
5. If `CHANGES REQUIRED` → send review feedback back to `Otter` with incremented iteration
6. Repeat until approved, max 5 iterations

If still not approved after 5 iterations, `Capybara` stops and surfaces remaining issues.

### Flow diagram

```mermaid
sequenceDiagram
  participant U as User
  participant C as Capybara
  participant S as Squirrel
  participant O as Otter
  participant W as Owl

  U->>C: Request

  alt Non-technical or simple
    C->>S: Delegate request
    S-->>C: Response
    C-->>U: Final response
  else Technical implementation
    loop Until APPROVED (max 5 iterations)
      C->>O: Implement task
      O-->>C: Implementation result
      C->>W: Request review
      W-->>C: APPROVED or CHANGES REQUIRED
    end
    C-->>U: Final reviewed result
  end
```

## Agent responsibilities

### Capybara (`Capybara.agent.md`)
- Orchestrates and routes requests
- For technical tasks, manages iterative Otter-Owl loop
- Forwards user prompt/context **verbatim** (no rewriting)
- Uses `#tool:agent/runSubagent` and `#tool:vscode/askQuestions`
- Has `disable-model-invocation: true` (pure router behavior)

### Otter (`Otter.agent.md`)
- Handles implementation/coding tasks
- Creates TODO plan, executes changes, runs available tests
- Writes implementation summary report:
  - `.github/temp_reports/{subfolder}/implementation_{iteration}.md`
- Focuses on practical, non-over-engineered solutions

### Owl (`Owl.agent.md`)
- Reviews technical output for correctness, completeness, quality
- Runs tests when available to detect regressions
- Classifies findings:
  - **Critical** → blocks approval (`CHANGES REQUIRED`)
  - **Minor** → suggestions only
- Writes review report:
  - `.github/temp_reports/{subfolder}/review_{iteration}.md`

### Squirrel (`Squirrel.agent.md`)
- Handles simple/general prompts:
  - explanations, brainstorming, casual Q&A, lightweight troubleshooting
- Does not modify code unless explicitly requested
- Keeps responses concise and helpful

## Fun facts 🐣

| Role          | Animal        | Why it fits |
|---------------|---------------|-------------|
| Router        | Capybara 🦫   | Calm, friendly, and sociable. Perfect energy for coordinating agents smoothly. |
| Implementer   | Otter 🦦      | Curious, playful, and clever. Skilled with tools and great at solving problems efficiently. |
| Reviewer      | Owl 🦉        | Classic symbol of wisdom and sharp observation. Spots sneaky issues and keeps everything in check. |
| Helper        | Squirrel 🐿️  | Quick, nimble, and full of energy. Always ready for quick support and small practical tasks. |

## How to use

- Start chat with `Capybara`.
- Ask naturally:
  - Technical request example: “Add endpoint X with validation and tests.”
  - Non-technical request example: “Explain this repository architecture.”
- `Capybara` routes automatically.
- For technical tasks, reports are generated under `.github/temp_reports/` per iteration.

## Report artifacts

During technical tasks, expect:
- `implementation_{iteration}.md` (from `Otter`)
- `review_{iteration}.md` (from `Owl`)

inside:
- `.github/temp_reports/{timestamp_objective}/`

This creates a lightweight audit trail of what was implemented and what was reviewed.

## Customization

Common tweaks you can make:
- Change models in frontmatter (`model:`)
- Restrict/expand tool access (`tools:`)
- Adjust review strictness in `Owl`
- Change max loop policy in `Capybara`
- Adapt tone/style prompts

## Official references

- https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-custom-agents
- https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents
- https://docs.github.com/en/copilot/reference/custom-agents-configuration

## License

MIT (see `LICENSE`).
