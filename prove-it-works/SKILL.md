---
name: prove-it-works
description: Verify that an AI-made change actually works before accepting "done". Use after any AI coding task, bug fix, or feature, whenever the AI says it is finished, fixed, passing, or working.
---

# Prove It Works

AI assistants often say a task is done when it is not. This skill turns "done" into evidence.

## When to use

Run this after every AI task, before you accept the result, merge, or move on. Works for any language, framework, or project type.

## Steps

1. **Write the claims down.** Restate what the AI said it did as a short list of checkable claims. Example: "Login now rejects an expired token," "the CSV export includes the date column," "the build passes."

2. **Detect how the project checks itself.** Look for the project's own verification commands. Common places:
   - Manifests and scripts: `package.json` scripts, `pyproject.toml`, `setup.cfg`, `Makefile`, `justfile`, `Taskfile.yml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `*.csproj`, `composer.json`, `Gemfile`
   - CI config: `.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.circleci/`
   - Docs: `README`, `CONTRIBUTING`

   Run the build, the test suite, the linter, and the type checker that the project uses. If none exist, say so. That is a finding, not a pass.

3. **Reproduce the original problem (for bug fixes).** Before the fix, the bug must be reproducible. After the fix, the same steps must succeed. If you cannot reproduce it before the change, you cannot prove the change fixed it.

4. **Exercise the changed behavior end to end.**
   - Library or function: call it with a normal input and at least one edge or failure input.
   - API or service: start it and send a real request. Check status code and response body.
   - CLI: run the command with real arguments and check the output and exit code.
   - UI: run the app and walk through the changed screen. Use the built-in browser or a screenshot when available.
   - Data or scripts: run on a small sample and inspect the output file.

5. **Check for silent regressions.** Re-run the tests that were green before the change. Read the diff for code that was deleted, commented out, or wrapped in a broad `try/catch` or `except` that swallows errors.

6. **Report with a verdict per claim.**

   | Claim | Verdict | Evidence |
   |---|---|---|
   | Expired tokens are rejected | VERIFIED | `curl` with expired token returned 401; output pasted |
   | Build passes | VERIFIED | `npm run build` exit 0 |
   | CSV includes date column | NOT VERIFIED | Did not run the export; need sample data |

   Verdicts: **VERIFIED** (you ran it and saw the result), **FAILED** (ran it and it broke), **NOT VERIFIED** (could not check, and say why).

## Rules

- Do not accept "should work", "looks correct", or "I updated the code" as evidence.
- Do not edit or delete tests to make them pass. If a test is wrong, say so and ask.
- If a check cannot run (missing credentials, no database, no network), report it as NOT VERIFIED with the reason. Never mark it passed.
- Show the command you ran and the relevant output. Keep excerpts short.
- If anything FAILED, the task is not done. Fix it or report it first.
