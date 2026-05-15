# PI Workstation — Workspace Instructions

## Purpose
This is the **PI Agent's workspace**. You are an autonomous coding agent. Treat this directory as your sandbox and command center.

## What You Can Do
- **Create files, directories, projects, scripts, tools — whatever you need.**
- **Experiment freely.** You have no restrictions on what you build here.
- **Write code, test code, debug, build, deploy prototypes** — all expected.

## Managing Other Repositories
- If you need to work on another repository, **add it as a submodule** rather than copying it in.
  ```bash
  git submodule add <url> <path>
  ```
- This keeps the workspace clean, version-controlled, and easy to update.

## Pushing
- **Commit your work regularly.** Write meaningful commit messages.
- **Push to the remote after each meaningful change** so progress is not lost.
- If a push fails, diagnose and fix the issue before continuing.

## Workflow
1. Read and follow instructions (like this file)
2. Execute tasks — coding, debugging, prototyping, research
3. Document decisions in `README.md` or inline comments
4. Commit and push frequently

## Dev Server
- **Always (re)start the Vite dev server after finishing a change.**
- **Always run in dev mode** (`import.meta.env.DEV` will be `true`).
- Kill any existing server on port 9000, then start a fresh one.
- This ensures the browser reflects the latest code, especially after CSS or layout changes.
- Example: `fuser -k 9000/tcp 2>/dev/null; sleep 2; cd /home/pi-agent/pi-workspace/school-of-thought && npx vite --port 9000 --host 0.0.0.0 &`
  - `npx vite` defaults to dev mode. Never use `--mode production` for the dev server.
