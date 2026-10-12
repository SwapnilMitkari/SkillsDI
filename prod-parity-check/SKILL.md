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
   | Unpinned versions | `flask` or `requests` with no version in `requirements.txt`; `latest` tags in Dockerfiles | Builds change without any code change |
   | Case-sensitive paths | Imports or file names whose case does not match on disk | Works on Windows or macOS, fails on Linux |
   | OS-specific code | Hardcoded `\` path separators, `C:\` paths, shell commands that exist only on one OS | Fails on Linux containers |
   | CORS and origins | Allowed origins set to `*` (`origins="*"`, `CORS_ALLOW_ALL_ORIGINS = True`, `Access-Control-Allow-Origin: *`, `@CrossOrigin(origins = "*")`) | Browser requests fail, or are open to any site |
   | Cookies and HTTPS | Cookies without `Secure` or `SameSite`; hardcoded `http://` links on an `https://` page (mixed content) | Login breaks behind HTTPS; browser blocks assets |
   | Health check | Deploy config expects an endpoint or probe that the app does not serve | Deploy rolls back |
   | Timezones and locale | Code assumes the server's local timezone or a fixed locale | Wrong dates in production |
   | Static assets | Asset URLs or paths that only resolve in dev; no build step for CSS or JS bundles | Broken styles, scripts, or images |
   | Third-party sandboxes | Test keys, sandbox URLs, or test webhook secrets in production config | Payments or emails go nowhere |
   | `.dockerignore` | `COPY . .` with no `.dockerignore` that excludes `.env`, `.git`, key files, and local data | Secrets and local data end up in the image |

   **B. Stack-specific checks**

   | Stack | Check | What to look for |
   |---|---|---|
   | Python (general) | Debug and runtime mode | `DEBUG = True`, `debug=True`, `FLASK_DEBUG=1`, `FLASK_ENV=development` |
   | Django | Settings for production | `DEBUG = True`; `ALLOWED_HOSTS = ['*']` or missing; `CORS_ALLOW_ALL_ORIGINS = True`; `SECRET_KEY` read from code instead of env; `STATIC_ROOT` missing before `collectstatic`; `MEDIA_ROOT` inside the project folder; `manage.py runserver` in the Dockerfile or `Procfile` (use gunicorn or uwsgi) |
   | Flask | Server and config | `app.run(debug=True)` or `app.run(host="127.0.0.1")`; `app.secret_key = "..."` as a literal; `flask run` or `python app.py` as the production command (use gunicorn or waitress); `CORS(app, origins="*")` |
   | Java (Spring, Maven, Gradle) | Profile and config | `SPRING_PROFILES_ACTIVE=dev` or `spring.profiles.active=dev` in production; `application.properties` or `application.yml` with localhost URLs; `server.port` fixed; `System.getenv` with no default; `JAVA_TOOL_OPTIONS` debug flags (`-agentlib:jdwp`); Dockerfile `CMD` with `mvn spring-boot:run` (use the built jar) |
   | Java (JDBC) | Connection string | `jdbc:...://localhost` or a password in the query string (`?password=`) |
   | Node.js | Dev command | `npm run dev`, `nodemon`, `ts-node` in production `CMD`; `NODE_ENV=development` |
   | Front end (HTML, CSS, JS) | Client build | Dev server or unminified bundles deployed; `http://localhost` URLs in HTML, CSS (`url(...)`), or JS; `console.log` of user data or secrets; `eval(` on untrusted input; source maps published with secrets inside; `https` page loading `http` scripts or styles |
   | IBM ESQL and App Connect | Deployment config | Broker or integration server properties pointing at a local MQ queue manager or database (`localhost`, `127.0.0.1`); flow properties with dev default values in `*.properties` override files; `DECLARE` with a dev URL instead of a property; missing overrides for production environment |
   | Serverless or PaaS | Fixed runtime | Writes to the local disk; long-running background threads or loops that the platform will kill; no `PORT` read |

3. **Check the deploy path itself.** Confirm the Dockerfile or build script copies the files the app needs and does not copy secrets or `.env` files into the image. Confirm the start command matches what the app expects in production. Check that each command the start line uses is declared in the manifest.

4. **Report.** Use a table sorted by severity:

   | Severity | Location | Problem | Fix |
   |---|---|---|---|
   | Blocker | `config/db.js:12` | Database URL hardcoded to `localhost` | Read from `DATABASE_URL` env var |
   | Blocker | `settings.py:2` | `DEBUG = True` in the production settings | Read from env with a default of `False` |
   | Warning | `Dockerfile:8` | Runs `npm run dev` | Use `npm start` after `npm run build` |
   | Warning | `requirements.txt` | `requests` imported but not listed | Add it, with a pinned version |

   - **Blocker**: production will fail or leak data. Do not deploy.
   - **Warning**: likely a problem or a risk. Fix before or right after deploy.
   - **Note**: a difference that is probably fine. State why.

## Rules

- Quote file and line for every finding.
- Do not change production config without asking. Propose the diff.
- If no deploy config exists, say so, and list the assumptions you made about the production environment.
- If a stack is not listed above, apply section A and check its own production-mode settings (the framework's documentation names them).
