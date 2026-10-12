---
name: prod-parity-check
description: Find code that works on a laptop but will break once deployed to production. Use before every deploy, release, or push to a production branch. Works across languages, including Python (Django, Flask), Java (Spring, Maven, Gradle), Node.js, IBM ESQL and App Connect, and front-end (HTML, CSS, JavaScript), in any hosting setup.
---

# Prod Parity Check

Local development hides many problems. This skill checks the code and deploy config for the differences that break production.

## When to use

Run before every deploy or release. It is most useful after the AI added new config, new environment variables, new file writes, new external URLs, or new dependencies.

## Steps

1. **Identify the target environment.** Find how the app is deployed: Dockerfiles, `docker-compose*.yml`, Kubernetes manifests (`k8s/`, `helm/`), `serverless.yml`, `vercel.json`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`, `app.yaml`, Terraform (`*.tf`), CI deploy jobs, platform config, and for ESQL the BAR file or integration server deployment scripts. Note what runs in production: which command starts the app, which OS image, which port.

   If there is no manifest at all (no `package.json`, `requirements.txt`, `pom.xml`, `build.gradle`, or equivalent), say so and check only what the code itself shows.

   If there is no deploy config at all, assume these production defaults and list each one in the report: the Dockerfile `CMD` if one exists; otherwise the `Procfile`; otherwise one process that listens on the `PORT` variable. Then run the checks against those assumptions. The assumptions themselves create no findings. A check that fails only because of an assumption is a Note, not a Blocker.

2. **Run the checks below.** Use search tools (grep or ripgrep) across the whole repo. Skip `node_modules`, `vendor`, `target`, `build`, `dist`, virtualenvs, and build output.

   **A. Checks for every stack**

   | Check | What to look for | Why it breaks |
   |---|---|---|
   | Environment variables | Every env var read in code must exist in the deploy config, with a documented default or a clear startup error. Search: `process.env`, `os.environ`, `os.getenv`, `System.getenv`, `@Value("${`, `getenv(`, `os.Getenv`, `settings.py` env reads | Missing var crashes or silently uses a dev value |
   | Hardcoded local addresses | `localhost`, `127.0.0.1`, `0.0.0.0`, `:3000`, `:8080` in code, HTML, CSS, JS, or config outside tests | Production cannot reach them |
   | Hardcoded ports | `listen(`, `app.run(`, `server.port`, `runserver 0.0.0.0:8000`, `System.out.println("port "...)` with a fixed number. The app should read the platform's `PORT` variable | Platform cannot route traffic |
   | Local file writes | Relative paths (`./uploads`, `./data`), `new File(`, `open(` with a relative path, `MEDIA_ROOT` pointing into the project, local SQLite files, logs to disk | Containers and serverless have read-only or ephemeral disks |
   | Database target | Connection strings pointing at dev, staging, or a local DB; missing SSL setting | Wrong data, or connection refused |
   | Migrations | Schema changes with no migration file, or migrations not run by the deploy step | App errors on missing columns |
   | Lockfiles | Dependency manifest changed but lockfile not updated; no lockfile at all (`package-lock.json`, `yarn.lock`, `poetry.lock`, `Pipfile.lock`, `go.sum`, `Cargo.lock`, `gradle.lockfile`) | Different versions in production |
   | Declared dependencies | Every import must be declared in the manifest (`requirements.txt`, `pyproject.toml`, `package.json`, `pom.xml`, `build.gradle`). Every command the deploy runs (for example `nodemon`, `gunicorn`) must be declared too | Works locally because the package is installed by hand, fails in the container |
   | Unpinned versions | `flask` or `requests` with no version in `requirements.txt`; `latest` tags in Dockerfiles. Floating tags such as `node:20` or `python:3.12` are a Note: they pick up patch releases, which is usually acceptable but should be a deliberate choice. | Builds change without any code change |
   | Outbound calls without timeout | HTTP or database calls with no timeout set (`requests.post(url)` with no `timeout=`, `fetch` with no abort, a client with no connect or read timeout) | A slow upstream hangs workers until the platform kills them |
   | Third-party scripts and styles | `<script src>` or `<link href>` loading from another origin with no `integrity` attribute | A changed or compromised CDN file runs on your page |
   | Case-sensitive paths | Imports or file names whose case does not match on disk | Works on Windows or macOS, fails on Linux |
   | OS-specific code | Hardcoded `\` path separators, `C:\` paths, shell commands that exist only on one OS | Fails on Linux containers |
   | CORS and origins | Allowed origins set to `*` (`origins="*"`, `CORS_ALLOW_ALL_ORIGINS = True`, `Access-Control-Allow-Origin: *`, `@CrossOrigin(origins = "*")`) | Browser requests fail, or are open to any site |
   | Cookies and HTTPS | Cookies without `Secure` or `SameSite`; hardcoded `http://` links on an `https://` page (mixed content) | Login breaks behind HTTPS; browser blocks assets |
   | Health check | Deploy config expects an endpoint or probe that the app does not serve | Deploy rolls back |
   | Timezones and locale | Code assumes the server's local timezone or a fixed locale | Wrong dates in production |
   | Static assets | Asset URLs or paths that only resolve in dev; no build step for CSS or JS bundles | Broken styles, scripts, or images |
   | Third-party sandboxes | Test keys, sandbox URLs, or test webhook secrets in production config | Payments or emails go nowhere |
   | `.dockerignore` | `COPY . .` with no `.dockerignore` that excludes `.env`, `.git`, key files, and local data | Secrets and local data end up in the image |
   | EXPOSE and port | Dockerfile `EXPOSE` differs from the port the app listens on, or from the `PORT` variable | The platform routes traffic to a closed port |
   | Ignore rules | `.gitignore` missing, or lacking env, key, keystore, and build folders | Secrets and build output get committed (see `leaked-key-finder`) |

   **B. Container build path (for any Dockerfile)**

   | Check | What to look for | Why it breaks |
   |---|---|---|
   | Working directory | `COPY` or `CMD` uses relative paths (`app.py`, `target/app.jar`, `manage.py`) but there is no `WORKDIR` line | The start command runs from `/` and cannot find the files |
   | Install step | The image never installs from the manifest (no `pip install -r`, `npm ci` or `npm install`, `mvn package`, `gradle build`, or equivalent) | Imports fail at startup |
   | Start target exists | Every file or artifact named in `CMD` or `ENTRYPOINT` is produced or copied into the image (for example `target/app.jar` is never built in the Dockerfile, or `manage.py` is missing from the project) | The container exits on start |
   | Base image | `latest` tags, or an image whose runtime version differs from the one the code was written for | Builds change without a code change |

   **C. Stack-specific checks**

   | Stack | Check | What to look for |
   |---|---|---|
   | Python (general) | Debug and runtime mode | `DEBUG = True`, `debug=True`, `FLASK_DEBUG=1`, `FLASK_ENV=development` |
   | Django | Settings for production | `DEBUG = True`; `ALLOWED_HOSTS = ['*']` or missing; `CORS_ALLOW_ALL_ORIGINS = True`; `SECRET_KEY` read from code instead of env; `STATIC_ROOT` missing before `collectstatic`; `MEDIA_ROOT` inside the project folder; `manage.py runserver` in the Dockerfile or `Procfile` (use gunicorn or uwsgi); no `urls.py` that routes to the views |
   | Flask | Server and config | `app.run(debug=True)` or `app.run(host="127.0.0.1")`; `app.secret_key = "..."` as a literal; `flask run` or `python app.py` as the production command (use gunicorn or waitress); `CORS(app, origins="*")` |
   | Java (Spring, Maven, Gradle) | Profile and config | `SPRING_PROFILES_ACTIVE=dev` or `spring.profiles.active=dev` in production; `application.properties` or `application.yml` with localhost URLs; `server.port` fixed; `System.getenv` with no default; `JAVA_TOOL_OPTIONS` debug flags (`-agentlib:jdwp`); Dockerfile `CMD` with `mvn spring-boot:run` (use the built jar) |
   | Java (JDBC) | Connection string | `jdbc:...://localhost` or a password in the query string (`?password=`) |
   | Node.js | Dev command | `npm run dev`, `nodemon`, `ts-node` in production `CMD`; `NODE_ENV=development` |
   | Front end (HTML, CSS, JS) | Client build | Dev server or unminified bundles deployed; `http://localhost` URLs in HTML, CSS (`url(...)`), or JS; `console.log` of user data or secrets; `eval(` on untrusted input; source maps published with secrets inside; `https` page loading `http` scripts or styles |
   | IBM ESQL and App Connect | Deployment config | Broker or integration server properties pointing at a local MQ queue manager or database (`localhost`, `127.0.0.1`); flow properties with dev default values in `*.properties` override files; `DECLARE` with a dev URL instead of a property; missing overrides for production environment. **Override binding:** an override line such as `OrderFlow#geoUrl=...` applies only if the name after `#` matches a property the flow declares. An override that binds to nothing is a Warning ("override binds to nothing"), because production silently keeps the default. A declared property is a name set by `DECLARE` or `SET` in the module's `.esql` file, or defined in the `.msgflow` file for that node. |
   | Serverless or PaaS | Fixed runtime | Writes to the local disk; long-running background threads or loops that the platform will kill; no `PORT` read |

   **D. Cross-references (do not repeat these here)**
   - Hardcoded secrets, keys, passwords, and secrets written to logs: run `leaked-key-finder`.
   - Loops, timers, retries, and public endpoints that call paid services: run `surprise-bill-guard`.
   - Failing unit tests: run `prove-it-works`.
   - Secrets in git history (committed, then removed): run `leaked-key-finder` step 3.
   - If a referred skill is not loaded in this environment, report "Cannot check: skill not loaded" against that item. Do not silently refer it.

3. **Assign a severity to every finding.** Use these defaults, so the same project gets the same ranking every time.

   | Severity | Checks that default to it |
   |---|---|
   | Blocker | Hardcoded production-breaking address (`localhost` DB or listen); `DEBUG` on in production; dev command in the production `CMD`; missing `WORKDIR` or dependency install in the Dockerfile; a `CMD` target that the image does not contain; a required env var read with no default; a live secret in client code (also refer to `leaked-key-finder`) |
   | Warning | Missing `.dockerignore` or `.gitignore` coverage; no lockfile; unpinned dependencies; `eval()` on input; CORS `*`; outbound calls with no timeout; third-party scripts with no `integrity`; mixed content; port or `EXPOSE` mismatch; override that binds to nothing |
   | Warning | Local file writes; hardcoded path separators; a `SECRET_KEY` literal (refer the secret to `leaked-key-finder`) |
   | Note | Floating image tags; unwired code (with the missing piece named); a default assumption that is probably fine; a documented example value |

   Raise a Warning to Blocker if it is on the production start path and breaks the first request.

   **Unwired code** is a Note for parity purposes, even if it contains an unbounded loop. Its cost risk is reported by `surprise-bill-guard`, not here.

   **No dependencies declared:** the lockfile and `.dockerignore` checks do not apply. Record each as a Note: "not applicable: no dependencies".

3b. **Check the deploy path itself.** Confirm the Dockerfile or build script copies the files the app needs and does not copy secrets or `.env` files into the image. Confirm the start command matches what the app expects in production, and that each command it uses is declared in the manifest.

4. **Find code that is never started.** For each route, job, or module that you flagged, check whether anything imports or starts it. What counts as an entry point, per stack:

   | Stack | Entry point that starts the code |
   |---|---|
   | Python | A `__main__` block, a `manage.py` command, a route registered in a `urls.py` or `app.route` decorator, or a scheduler call |
   | Java | A `main` method, a `@Scheduled` method, or a `scheduleAtFixedRate` call reached from `main` |
   | Node.js | The `start` script in `package.json`, or a `require` chain from the file it names |
   | IBM ESQL | A flow whose input node connects to the node in question (check the `.msgflow` wiring) |
   | Front end | A `<script>` tag or a page that loads the file |

   If nothing starts the code, mark it **unwired** and say what would start it. If something tries to start it but a piece is missing (for example, a URL config with no `urls.py`, or a `manage.py` that does not exist), mark it **unwired: missing piece** and name the missing file. Unwired code is not a production blocker by itself, but it can become one when someone wires it up.

5. **Report.** Use a table sorted by severity. Add a separate list for anything you could not check, with the reason.

   | Severity | Location | Problem | Fix |
   |---|---|---|---|
   | Blocker | `config/db.js:12` | Database URL hardcoded to `localhost` | Read from `DATABASE_URL` env var |
   | Blocker | `settings.py:2` | `DEBUG = True` in the production settings | Read from env with a default of `False` |
   | Blocker | `Dockerfile:4` | `CMD ["java", "-jar", "target/app.jar"]` with no `WORKDIR` and no build step that creates `target/app.jar` | Add `WORKDIR` and a build stage |
   | Warning | `Dockerfile:8` | Runs `npm run dev` | Use `npm start` after `npm run build` |
   | Warning | `requirements.txt` | `requests` imported but not listed | Add it, with a pinned version |
   | Note | `shop/views.py:12` | View is never routed (no `urls.py`) | Unwired; add a URL config or remove the view |

   - **Blocker**: production will fail to start or run, leak data, or run up unbounded cost. Do not deploy.
   - **Warning**: likely a problem or a risk. Fix before or right after deploy.
   - **Note**: probably fine, or unwired code. State why, and say what would start it.
   - **Cannot check**: a check that needs a tool, a running service, or a file the project does not have. Give the reason.

## Rules

- Quote file and line for every finding.
- Do not change production config without asking. Propose the diff.
- If no deploy config exists, say so, and list the assumptions you made about the production environment.
- If a stack is not listed above, apply section A and check its own production-mode settings (the framework's documentation names them).
- Do not run the project to check it. This skill is read-only analysis.
