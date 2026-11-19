# AI Dev Orchestrator

Requirements in, reviewed code out. Agents do all the development; a human only reviews.

```
Brief ──► Planner agent ──► Task board ──► [Start]
                                             │
             ┌───────────────────────────────┘
             ▼
   Backend agent (backend/) ──► Frontend agent (frontend/) ──► Tester agent (browser)
             ▲                         ▲                              │
             └──── failures routed to the agent that owns them ◄──────┤ (auto, max 2 retries)
                                                                      ▼
                                                    Developer review (human)
                                       Approve & merge │ Request changes → child task
                                                       │   for backend / frontend / both
```

| Agent | Can do | Can't do |
|---|---|---|
| Planner | Read the repo; write the PRD, tasks, API contract and acceptance criteria | Edit files |
| Backend dev | Edit files in `backend/` only, run commands | Touch `frontend/` |
| Frontend dev | Edit files in `frontend/` only, run commands | Touch `backend/` |
| Tester | Drive the running app in a real browser (Playwright MCP), take screenshots | Read or edit code |
| **You** | Answer the planner's questions, click Start, review each agent's diff and the test report, approve or request changes | Write code |

## Run it

Requires Node 22+, git, and either `ANTHROPIC_API_KEY` or a Claude Code login on this machine.

```bash
cd ~/Desktop/ai-dev-orchestrator
cp .env.example .env          # add ANTHROPIC_API_KEY, or leave blank if you're logged in to Claude Code
npm install
npm run setup:demo            # creates workspace/demo-app, the repo the agents work on
npm run dev                   # API on :4000, UI on http://localhost:5173
```

Then in the UI:

1. Write a brief, e.g. *"Users can add todos, mark them done and delete them. Todos are stored by the backend."*
2. Answer the planner's clarifying questions → tasks appear in **Backlog**.
3. Click **Start** on a task. Watch the backend agent, then the frontend agent, then the tester (a browser window opens).
4. In **Your review**, open the task: check the **Backend agent diff**, the **Frontend agent diff**, the **Tester report** (with screenshots) and the **Live preview**.
5. **Approve & merge**, or **Request changes** and pick which agent gets them. The change request becomes a new card; click Start on it.

## Use your own app

Set `TARGET_REPO` to any git repo whose `npm run dev` starts the app using the `PORT` (UI) and `API_PORT` (backend)
environment variables. Set `FRONTEND_DIR` / `BACKEND_DIR` to the folders each agent owns, and `PREVIEW_COMMAND`
if the start command is different.

## How it works

- **One git worktree + branch per task** (`workspace/worktrees/task-N`, branch `task/N-…`), so parallel tasks don't collide.
  Change requests reuse their parent's branch.
- **Each agent's changes are separate commits** authored as "Backend Agent" / "Frontend Agent", so you can review them separately.
- **Edits are fenced by folder**: a `canUseTool` hook denies writes outside the agent's own folder.
- **The tester tags every failure** as `backend`, `frontend` or `unknown`; only the responsible agent is re-run.
- **Guard rails**: `MAX_TURNS_*`, `MAX_BUDGET_USD_PER_RUN`, `MAX_TEST_RETRIES`, and a **Stop agents** button.
- State lives in SQLite (`data/orchestrator.db`); live updates go to the UI over Server-Sent Events.

```
server/src
  index.ts          REST API + SSE
  orchestrator.ts   state machine: requirement → tasks → backend → frontend → tester → review
  workspace.ts      git worktrees, per-agent commits and diffs, preview servers, merge
  agents/
    runAgent.ts     Claude Agent SDK wrapper (streams activity to the UI)
    planner.ts      clarifying questions, then PRD + tasks (structured JSON output)
    developer.ts    backend + frontend dev agents, each fenced to its folder
    tester.ts       browser tester using Playwright MCP
web/src             React UI: intake panel, board, task drawer, review bar
examples/demo-target  starter full-stack app copied to workspace/demo-app
```
# orchestretor
