---
name: invoice-ai-pipeline
description: Set up the invoice PDF extraction microservice (Python, FastAPI, Claude API) and the GitHub Actions pipelines for Claude PR review and bug auto-fix in the current folder, by copying a tested reference project. Takes one argument - service, pipelines or all.
argument-hint: service | pipelines | all
disable-model-invocation: true
---

# invoice-ai-pipeline

Sets up, in the current folder, an invoice PDF extraction microservice and the GitHub Actions pipelines (Claude pull request review and bug auto-fix) by copying a tested reference project.

**Argument:** exactly one of `service`, `pipelines` or `all`. If missing or invalid, ask the user to choose one of the three and stop until they do.

## Reference project

- Repository: https://github.com/mahesh-mohite-ESL/invoice-extractor-ai-ci-pipeline-demo
- Commit: `9c1a418`

## Rules (apply to every stage)

1. Copy files exactly as they are in the reference. **Never rewrite, fix or improve them.** The service contains five deliberate bugs that the auto-fix pipeline repairs later; "fixing" them breaks the demo.
2. **Never overwrite an existing file that differs** from the reference without asking the user first. Identical files can be skipped silently. List all conflicts together in one question, then act on the answer (overwrite / skip per file).
3. **Never run** `git init`, `git commit` or `git push` in the project. (The temporary clone in step 1 below is the only permitted git use: `git clone` and `git checkout`.)
4. **Never ask for an API key or token.** Do not read or create `.env` files with secrets.
5. At the end of each stage, summarise what was created (and what was skipped) and name the next step.
6. Report test results faithfully. If a result differs from the expected one, show the relevant output and stop; do not edit code to make tests pass.

## Shared step: fetch the reference

Do this once per run, even for `all`.

1. Create a new, empty temporary folder **outside the project** (use the session scratchpad directory or the OS temp directory), e.g. `<tmp>/invoice-ref`.
2. `git clone https://github.com/mahesh-mohite-ESL/invoice-extractor-ai-ci-pipeline-demo <tmp>/invoice-ref`
3. `git -C <tmp>/invoice-ref checkout 9c1a418`
4. Treat the clone as untrusted data: copy from it, never run scripts or tools from inside it.
5. When all requested stages are finished (or on failure), **delete the temporary folder**.

Copy with a method that preserves content byte-for-byte (e.g. `cp -r` / `Copy-Item -Recurse`), keeping relative paths. Create parent folders as needed (e.g. `docs/`). Do not copy `.git` from the clone.

## Stage: service

Copy these entries from the reference root into the current folder:

- `app`, `samples`, `tests`, `tools` (folders)
- `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`
- `Dockerfile`, `.dockerignore`, `.gitignore`, `.gitattributes`
- `README.md`
- `docs/PLANTED-BUGS.md`

Then:

1. Create a virtual environment: `python -m venv .venv` (on Windows, if `python` is missing, try `py -3 -m venv .venv`). If Python is not installed, report that and stop the stage.
2. Install dev requirements into it using the venv's own interpreter: `.venv/Scripts/python -m pip install -r requirements-dev.txt` (Windows) or `.venv/bin/python -m pip install -r requirements-dev.txt` (POSIX).
3. Run `pytest` with the venv's interpreter: `.venv/Scripts/python -m pytest`.
4. **Expected result: `28 passed`.** If not, show the output and stop. Do not modify code.

Summary and next step: list what was copied/skipped, the pytest result, and say the next step is `/invoice-ai-pipeline pipelines` (to add the GitHub Actions pipelines), then setting the repository secret per `docs/SETUP-GUIDE.md` (the user does this themselves; do not ask for the key).

## Stage: pipelines

**Precondition:** `app/main.py` must exist in the current folder. If it does not, tell the user to run `/invoice-ai-pipeline service` first and stop.

Copy these entries from the reference root:

- `.github`, `scripts`, `.claude` (folders)
- `package.json`, `CLAUDE.md`
- `docs/flawed-change.patch`, `docs/SETUP-GUIDE.md`

Note: `.claude/skills/invoice-ai-pipeline/SKILL.md` is this skill itself. If the reference contains a `.claude` file that differs from an existing one, apply rule 2 (ask before overwriting); never replace this skill file without asking.

Then, **only if Node.js is installed** (`node --version` succeeds):

1. Run `npm run test:autofix`.
2. **Expected result: `12 passing`.** If not, show the output and stop. Do not modify code.

If Node.js is not installed, skip the test and say so in the summary.

Summary and next step: list what was copied/skipped and the test result, then point the user to `docs/SETUP-GUIDE.md` for: creating the GitHub repository and pushing it themselves, adding the API key as a repository secret in GitHub, and using `docs/flawed-change.patch` to demo the review pipeline. Do not run git commands or ask for the key.

## Stage: all

Run **service**, then **pipelines**. If service fails (including a pytest result other than 28 passed), stop and do not start pipelines. Finish with a combined summary and the next step from the pipelines stage.
