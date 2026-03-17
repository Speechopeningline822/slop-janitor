# `codex-refactor-loop`

`codex-refactor-loop` is a small CLI that drives the Codex app-server through a repeatable multi-turn workflow.

The point is simple: if you use Codex directly, chaining a long sequence of messages is manual work. You ask for a refactor. Then you ask it to improve the plan. Then improve it again. Then implement it. Then review the result. Then review it again with fresh eyes. `codex-refactor-loop` automates that chain.

That matters because the value is not in any single turn. The value comes from compounding turns. A repo often gets materially better only after Codex has had a chance to propose a direction, sharpen it, implement it, and then critique its own work. `codex-refactor-loop` keeps that whole process on one thread so each step can build on the previous one instead of starting over.

The practical promise is straightforward: run it on a repository, especially in refactor mode, and it should push the codebase toward something cleaner, simpler, and more reliable than you usually get from a one-shot prompt. It also writes a complete run log, so the session is inspectable after the fact rather than something that only existed in the terminal.

By default, one cycle is:

1. `execplan-create`
2. `execplan-improve` x4
3. `implement-execplan`
4. `review-recent-work` x5

You can change the number of full cycles, improvement passes, and review passes.

## Why Use It

- It automates the long chain of Codex follow-up messages you would otherwise run by hand.
- It gives you compounding improvement instead of a single pass.
- It preserves continuity by reusing one thread across planning, implementation, and review.
- In refactor mode, it is explicitly trying to make the repo better and more reliable, not just produce one plausible patch.
- It stays noninteractive, so failures are explicit instead of hiding behind approval prompts.
- It keeps the terminal readable by showing only agent commentary, final agent text, and token usage.
- It writes a full timestamped log with stage banners, command output, file-change progress, MCP progress, and item lifecycle events.
- It works from the root of any repo because it sends your current working directory to `thread/start`.

This is useful when you want Codex to keep working the problem until the repo is in better shape, not just answer once.

## Prerequisites

- Python 3.11 or newer.
- Rust and `cargo`.
- A separate clone of the open-source Codex repository.
- ChatGPT authentication if the active Codex provider requires OpenAI auth.

The bundled skills used by `codex-refactor-loop` live in `.agents/skills` inside this repository.

## Setup

Clone this repository and clone Codex separately:

```bash
git clone https://github.com/grp06/codex-refactor-loop.git
git clone https://github.com/openai/codex.git
```

Point `codex-refactor-loop` at the Codex Rust workspace:

```bash
export CODEX_WORKSPACE=/path/to/codex/codex-rs
```

You can also pass the path per command with `--codex-workspace /path/to/codex/codex-rs`.

Authenticate through the wrapped Codex login flow:

```bash
cd codex-refactor-loop
./codex-refactor-loop auth login
./codex-refactor-loop auth login --device-auth
./codex-refactor-loop auth status
./codex-refactor-loop auth logout
```

The auth wrapper keeps stdin, stdout, and stderr attached to the terminal, so browser and device-code login behave like native `codex login`.

If you sign in with your ChatGPT account, Codex uses the access included with your ChatGPT plan. OpenAI's current docs say Codex is included with ChatGPT Plus and Pro, so the normal ChatGPT-authenticated flow uses that plan-backed access for inference instead of requiring you to paste an API key manually. Official docs: [Using Codex with your ChatGPT plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).

## Basic Use

The most natural use is refactor mode. Run it from the repository you want to improve:

```bash
cd /path/to/target-repo
/path/to/codex-refactor-loop/codex-refactor-loop --mode refactor
```

Add guidance if you want to steer the refactor:

```bash
cd /path/to/target-repo
/path/to/codex-refactor-loop/codex-refactor-loop --mode refactor --prompt "focus on testability and simplifying boundaries"
```

Run the default planning-first workflow if you want to start from an open-ended implementation prompt instead:

```bash
cd /path/to/target-repo
/path/to/codex-refactor-loop/codex-refactor-loop --prompt "help me build a CRM"
```

Increase the amount of iteration:

```bash
cd /path/to/target-repo
/path/to/codex-refactor-loop/codex-refactor-loop --prompt "help me build a CRM" --cycles 2 --improvements 5 --review 3
```

`codex-refactor-loop` always targets the directory you launch it from, not the `codex-refactor-loop` repository.

## Modes And Counts

`--mode pipeline` is the default. It requires `--prompt` and starts with `execplan-create`.

`--mode refactor` keeps the same follow-up structure, but replaces stage 1 with `find-best-refactor`. `--prompt` is optional in refactor mode. If you omit it, the first stage asks for the single highest-leverage refactor in the current repository.

`--cycles` controls how many times the full loop runs.

`--improvements` controls how many `execplan-improve` turns run inside each cycle.

`--review` controls how many `review-recent-work` turns run inside each cycle.

Defaults:

- `--cycles 1`
- `--improvements 4`
- `--review 5`

When `--cycles` is greater than 1, stage labels in the run log are cycle-qualified, for example `cycle-2-execplan-create`.

## Codex Workspace Configuration

When `codex-refactor-loop` launches the real Codex app-server or wrapped auth commands, it resolves the Codex workspace in this order:

1. `--codex-workspace /path/to/codex-rs`
2. `CODEX_WORKSPACE`

If neither is set, the command fails with a clear setup error.

Examples:

```bash
./codex-refactor-loop --codex-workspace /path/to/codex/codex-rs --prompt "help me build a CRM"
./codex-refactor-loop auth --codex-workspace /path/to/codex/codex-rs login
```

## What It Actually Does

Before stage 1, the client performs:

1. `initialize` with `capabilities.experimentalApi = true`
2. `initialized`
3. `account/read`
4. `thread/start`

If `account/read` says OpenAI auth is required and no account is logged in, the command fails immediately and tells you to run `./codex-refactor-loop auth login`.

After that, every stage runs as a `turn/start` on the same thread. That is what gives the workflow continuity. The implementation and review stages see the plan that was just created and improved.

## Output Model

The terminal is intentionally sparse. During a run, it shows:

- agent-message commentary
- final agent-message text
- token usage

Everything else goes to the run log:

- stage banners
- command output
- file-change progress
- MCP progress
- item lifecycle notices
- failure details

Each run writes a full log to `runs/`. Log filenames start with the basename of the directory you launched from, followed by a UTC timestamp, for example `my-repo-20260317T213000Z.log`.

This split is deliberate. The terminal stays readable while the log remains complete.

## Reliability Contract

- Model and sandbox settings are inherited from your current Codex config. In v1, `codex-refactor-loop` only overrides `cwd` and `approvalPolicy`.
- The thread uses `approvalPolicy: "never"`.
- If the server asks for approvals, user input, permissions, MCP elicitation, or ChatGPT token refresh, `codex-refactor-loop` responds deterministically, marks the stage failed, and exits after the matching `turn/completed`.
- Successful turns require real token data from `thread/tokenUsage/updated`. If a turn completes successfully without token usage, the run fails instead of printing invented zeros.
- Skill paths are validated before the app-server starts, so broken local setup fails early.

The tool is strict on purpose. When something is wrong, it should stop in a way you can diagnose.

## Tests

Run the test suite from the repository root:

```bash
python3 -m unittest discover -s tests -p 'test_*.py' -v
```
