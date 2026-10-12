---
name: prove-it-works
description: Verify that an AI-made change actually works before accepting "done". Use after any AI coding task, bug fix, or feature, whenever the AI says it is finished, fixed, passing, or working. Works across languages, including Python, Java, Node.js, IBM ESQL, front-end (HTML, CSS, JavaScript), Django, and Flask.
---

# Prove It Works

AI assistants often say a task is done when it is not. This skill turns "done" into evidence.

## When to use

Run this after every AI task, before you accept the result, merge, or move on.

## Steps

1. **Write the claims down.** Restate what the AI said it did as a short list of checkable claims. Take the claims from, in order:
   - The user's request, if it names outcomes.
   - The AI's final message, if there is one.
   - If neither exists, the changed code and commit messages. Code comments and test names describe intent; they do not prove it, so they are not claims. Say that the claims were inferred, and list them.

   Never invent a claim that nothing supports. If you cannot find any claim, say so, run the project's own checks in step 2, and report "no claims to verify" with the list of files reviewed.

   Example claims: "Login now rejects an expired token," "the CSV export includes the date column," "the build passes."

2. **Detect the stack and its test runner.** Find the project's manifests and pick the matching row. Run the project's own commands. Do not install tools silently.

   | Stack | Detect by | Run | If the tool is missing |
   |---|---|---|---|
   | Python | `pyproject.toml`, `requirements.txt`, `setup.cfg`, `pytest.ini`, `tox.ini` | `pytest` or `python -m unittest` | Report NOT VERIFIED and show the install command |
   | Django | `manage.py` (must exist), `settings.py` | `python manage.py test` | If `manage.py` is absent, say so. Do not claim Django tests passed. |
   | Flask | `app.py` or `wsgi.py`, `requirements.txt`, `pyproject.toml` | `pytest` if present; otherwise none | If there are no tests, report "no tests" as a finding |
   | Node.js | `package.json` (`scripts.test`, `scripts.lint`, `scripts.build`) | `npm test`, `npm run lint`, `npm run build` | Report NOT VERIFIED if the script is missing |
   | Java (Maven) | `pom.xml` | `mvn -q test` (or `./mvnw -q test`) | Report NOT VERIFIED if `mvn` is missing |
   | Java (Gradle) | `build.gradle`, `build.gradle.kts`, `settings.gradle` | `./gradlew test` (or `gradle test`) | Report NOT VERIFIED if neither is available |
   | IBM ESQL (App Connect / IIB) | `*.esql`, `*.msgflow`, `*.subflow`, `*.application`, `*.bar` | The project's unit-test project, run with the IBM toolkit's build or `ibmint`/`mqsi` tools | This usually needs the IBM toolkit installed. Without it, report NOT VERIFIED. Never mark ESQL as passed by reading it. |
   | Front end (HTML, CSS, JS) | `package.json`, `.stylelintrc*`, `.eslintrc*`, `eslint.config.*`, `.htmlvalidate*` | `npm run lint`, `npx stylelint "**/*.css"`, `npx eslint .`, `npx html-validate "**/*.html"` if configured | With no lint or test config, report "no automated checks" as a finding. Review the changed files by hand, list each manual check you made and its limits, and mark any automated claim NOT VERIFIED |
   | Other | Makefile, justfile, Taskfile, CI config (`.github/workflows/`, `.gitlab-ci.yml`) | Use the target the CI runs | If nothing is found, say so. That is a finding, not a pass. |

   Also check CI config (`.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.circleci/`) for the exact commands the project uses.

   **Runner notes:**
   - **Tool missing vs project cannot build:** a missing tool is NOT VERIFIED. A project that cannot build with its own declared tools (a test imports a library its manifest does not declare, or a file it needs is absent) is FAILED, with the reason. Zero tests run is NOT VERIFIED, and it is a finding in its own right.
   - **Node.js on Node 24 (and on Windows):** `node --test <directory>` can fail with `MODULE_NOT_FOUND` before any assertion runs. Pass explicit test file paths instead (for example `node --test test/pricing.test.js`). Report a runner error as NOT VERIFIED.
   - **Python unittest:** if it reports `NO TESTS RAN` for pytest-style functions, the tests were not run. Report NOT VERIFIED. Install pytest and say so, or call the test functions directly and say that you did.

3. **Reproduce the original problem (for bug fixes).** Before the fix, the bug must be reproducible. After the fix, the same steps must succeed. If you cannot reproduce it before the change, you cannot prove the change fixed it. If there is no change set (a whole-project review), write "not applicable: no change set" and continue with step 4.

4. **Exercise the changed behavior end to end.**
   - Library or function: call it with a normal input and at least one edge or failure input.
   - Web API (Flask, Django, Spring, Express, and similar): start it and send a real request. Check status code and response body.
   - CLI: run the command with real arguments and check the output and exit code.
   - UI (HTML, CSS, JS): run the page and walk through the changed screen, using the built-in browser or a screenshot when available. Check the browser console for errors.
   - Message flow (ESQL): run the flow's unit tests with sample input messages and check the output message.
   - Data or scripts: run on a small sample and inspect the output file.

5. **Run the test command with a timeout, and read it carefully.**
   - Put a timeout on every test or server command. A hang is FAILED, with the reason "did not exit within N seconds". A test that starts a server must close it, or the runner will wait forever. Decide a hang by where it stopped: if the tests finished and the process did not exit, it is FAILED (hang). If the runner stopped before any test ran, it is NOT VERIFIED (runner error).
   - Tell runner errors apart from test failures. A `MODULE_NOT_FOUND`, a compile error, or a missing environment variable means the tests never ran. Report that as NOT VERIFIED, not as a passed or failed assertion.
   - If a required environment variable is missing, look for `.env.example`, the README, or the CI config for the documented values before running. Do not let an import-time crash hide the real result.
   - **Ports:** check that the port is free before you start a server. If it is taken by a process you did not start, do not stop it. Use another port through the app's `PORT` setting, or report NOT VERIFIED with the reason. If the app hardcodes its port (for example `listen(3000)`) and that port is taken, report NOT VERIFIED. Do not edit the project's port to get around it.
   - Stop any server or process you started during verification.

6. **Check whether running is safe.** Before you run code, check whether it calls external services, sends messages, charges money, or loops over paid calls.
   - If it does, do not run it against live services. Use test-mode keys or stubs only if the user provided them. Otherwise report NOT VERIFIED with the reason "skipped: would call an external service".
   - Never run a loop that makes paid calls to see what happens.
   - Browser pages that call paid APIs are checked by reading the code, not by running them.

7. **Scan for tests that cannot fail.** Search the test files for vacuous assertions and skipped tests:
   - Python: `assert True`, `self.assertTrue(True)`, `@pytest.mark.skip`, `@unittest.skip`
   - Java: `assertTrue(true)`, `@Disabled`, `@Ignore`
   - Node.js and front end: `assert.ok(true)`, `expect(true)`, `it.skip`, `test.skip`, `xit(`
   - Django: `self.assertTrue(True)`, `skipIf`
   - Any language: an empty test body, or a test that was edited to match the new output

   Report each one. A test that cannot fail does not prove anything.

8. **Check for silent regressions.** Re-run the tests that were green before the change. Read the diff for code that was deleted, commented out, or wrapped in a broad `try/catch`, `except`, `catch (Exception e)`, or `ON ERROR` handler (ESQL) that swallows errors.

   **Baseline:** if there is no record of which tests passed before the change, say "no baseline". Where possible, run the suite once before the change to record one. A whole-project review has no baseline by definition: say "no baseline" and skip the regression comparison.

9. **Report with a verdict per claim.**

   | Claim | Verdict | Evidence |
   |---|---|---|
   | Expired tokens are rejected | VERIFIED | `curl` with expired token returned 401; output pasted |
   | Build passes | VERIFIED | `npm run build` exit 0 |
   | CSV includes date column | NOT VERIFIED | Did not run the export; need sample data |
   | ESQL flow routes errors to the fault terminal | NOT VERIFIED | IBM toolkit not installed; no runner available |
   | Discount is 10% off 100 | FAILED | `node --test test/pricing.test.js`: actual 91, expected 90 |

   Verdicts: **VERIFIED** (you ran it and saw the result), **FAILED** (ran it and it broke, including a hang), **NOT VERIFIED** (could not check, and say why).

   Also list the claims source (step 1), the baseline status (step 8), and any port or safety skips (steps 5 and 6).

## Rules

- Do not accept "should work", "looks correct", or "I updated the code" as evidence.
- Do not edit or delete tests to make them pass. If a test is wrong, say so and ask.
- A file that cannot be run (no runner, no toolkit) is never PASS. Report NOT VERIFIED with the reason.
- Do not stop a process you did not start, even if it blocks a port.
- Show the command you ran and the relevant output. Keep excerpts short.
- If anything FAILED, the task is not done. Fix it or report it first.
