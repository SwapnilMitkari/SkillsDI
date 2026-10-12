---
name: prove-it-works
description: Verify that an AI-made change actually works before accepting "done". Use after any AI coding task, bug fix, or feature, whenever the AI says it is finished, fixed, passing, or working. Works across languages, including Python, Java, Node.js, IBM ESQL, front-end (HTML, CSS, JavaScript), Django, and Flask.
---

# Prove It Works

AI assistants often say a task is done when it is not. This skill turns "done" into evidence.

## When to use

Run this after every AI task, before you accept the result, merge, or move on.

## Steps

1. **Write the claims down.** Restate what the AI said it did as a short list of checkable claims. Example: "Login now rejects an expired token," "the CSV export includes the date column," "the build passes."

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
   | Front end (HTML, CSS, JS) | `package.json`, `.stylelintrc*`, `.eslintrc*`, `eslint.config.*`, `.htmlvalidate*` | `npm run lint`, `npx stylelint "**/*.css"`, `npx eslint .`, `npx html-validate "**/*.html"` if configured | With no lint or test config, report "no automated checks" as a finding and check manually |
   | Other | Makefile, justfile, Taskfile, CI config (`.github/workflows/`, `.gitlab-ci.yml`) | Use the target the CI runs | If nothing is found, say so. That is a finding, not a pass. |

   Also check CI config (`.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.circleci/`) for the exact commands the project uses.

3. **Reproduce the original problem (for bug fixes).** Before the fix, the bug must be reproducible. After the fix, the same steps must succeed. If you cannot reproduce it before the change, you cannot prove the change fixed it.

4. **Exercise the changed behavior end to end.**
   - Library or function: call it with a normal input and at least one edge or failure input.
   - Web API (Flask, Django, Spring, Express, and similar): start it and send a real request. Check status code and response body.
   - CLI: run the command with real arguments and check the output and exit code.
   - UI (HTML, CSS, JS): run the page and walk through the changed screen, using the built-in browser or a screenshot when available. Check the browser console for errors.
   - Message flow (ESQL): run the flow's unit tests with sample input messages and check the output message.
   - Data or scripts: run on a small sample and inspect the output file.

5. **Run the test command with a timeout, and read it carefully.**
   - Put a timeout on every test or server command. A hang is FAILED, with the reason "did not exit within N seconds".
   - Tell runner errors apart from test failures. A `MODULE_NOT_FOUND`, a compile error, or a missing environment variable means the tests never ran. Report that as NOT VERIFIED, not as a passed or failed assertion.
   - If a required environment variable is missing, look for `.env.example`, the README, or the CI config for the documented values before running. Do not let an import-time crash hide the real result.
   - Stop any server or process you started during verification. Check that the port is free before starting one.

6. **Scan for tests that cannot fail.** Search the test files for vacuous assertions and skipped tests:
   - Python: `assert True`, `self.assertTrue(True)`, `@pytest.mark.skip`, `@unittest.skip`
   - Java: `assertTrue(true)`, `@Disabled`, `@Ignore`
   - Node.js and front end: `assert.ok(true)`, `expect(true)`, `it.skip`, `test.skip`, `xit(`
   - Django: `self.assertTrue(True)`, `skipIf`
   - Any language: an empty test body, or a test that was edited to match the new output

   Report each one. A test that cannot fail does not prove anything.

7. **Check for silent regressions.** Re-run the tests that were green before the change. Read the diff for code that was deleted, commented out, or wrapped in a broad `try/catch`, `except`, `catch (Exception e)`, or `ON ERROR` handler (ESQL) that swallows errors.

8. **Report with a verdict per claim.**

   | Claim | Verdict | Evidence |
   |---|---|---|
   | Expired tokens are rejected | VERIFIED | `curl` with expired token returned 401; output pasted |
   | Build passes | VERIFIED | `npm run build` exit 0 |
   | CSV includes date column | NOT VERIFIED | Did not run the export; need sample data |
   | ESQL flow routes errors to the fault terminal | NOT VERIFIED | IBM toolkit not installed; no runner available |

   Verdicts: **VERIFIED** (you ran it and saw the result), **FAILED** (ran it and it broke), **NOT VERIFIED** (could not check, and say why).

## Rules

- Do not accept "should work", "looks correct", or "I updated the code" as evidence.
- Do not edit or delete tests to make them pass. If a test is wrong, say so and ask.
- A file that cannot be run (no runner, no toolkit) is never PASS. Report NOT VERIFIED with the reason.
- Show the command you ran and the relevant output. Keep excerpts short.
- If anything FAILED, the task is not done. Fix it or report it first.
