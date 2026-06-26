# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

NL2RepoBench is a benchmark that evaluates LLMs/coding agents on **0-to-1 long-horizon tasks**: generating a complete, runnable code repository from a natural-language spec. It contains 104 tasks, each with its own test environment, and drives OpenHands in headless batch mode to execute them.

## Common Commands

This project uses [uv](https://docs.astral.sh/uv/) for dependency management. The project is pinned to Python 3.12 (`.python-version`); dependencies are declared in `pyproject.toml` and locked in `uv.lock`. (`requirements.txt` is kept for reference but is no longer the source of truth.)

Install dependencies (creates `.venv/` and installs from `uv.lock`):

```bash
uv sync
```

Run the full benchmark (reads `config.json`, loads `test_files/*`, fans out to OpenHands containers):

```bash
uv run python main.py
```

Re-run just the post-processing/grading for a single existing workspace (no LLM call — edit `task_id` and `pro_name` at the bottom of the file first):

```bash
uv run python only_test.py
```

There is no automated test target for the harness itself. `tests/test_docker.py` is an ad-hoc Docker connectivity check, not a pytest suite.

## Linting

Linting and formatting are handled by [Ruff](https://docs.astral.sh/ruff/) (configured under `[tool.ruff]` in `pyproject.toml`: line length 100, rules `E`/`F`/`I`). It is installed as a dev dependency via `uv sync`.

Check for lint issues:

```bash
uv run ruff check .
```

Auto-fix the fixable subset (unused imports, import sorting, etc.):

```bash
uv run ruff check . --fix
```

Check / apply formatting:

```bash
uv run ruff format --check .   # report only
uv run ruff format .           # rewrite in place
```

Note: the existing harness code predates Ruff and currently reports a number of `E501` (line-too-long) and `F403`/`F405` (star-import) findings. These are pre-existing and not auto-fixed; clean them up incrementally rather than in one sweep.

## Prerequisites

Docker must be running locally with these images pulled:

- `docker.all-hands.dev/all-hands-ai/openhands:0.56`
- `docker.all-hands.dev/all-hands-ai/runtime:0.56-nikolaik`

Each per-task test image is pulled on demand from `ghcr.io/multimodal-art-projection/nl2repobench/<proName>:1.0` (see `openhands/post_processor.py:282`).

## Architecture

The benchmark is a three-stage pipeline orchestrated per task, with all stages running in parallel across tasks via a `ThreadPoolExecutor`.

**1. Task loading** (`test_data_service.py`)
Scans `test_files/*` once at startup. Each subdirectory is one task (its name == `proName` and must appear in `config.json`'s `proNameList`). Files are discovered by suffix/keyword, not fixed names:
- `*.txt` → expected test case count
- `*commands*.json` → shell commands to run inside the test image (typically `pytest ...`)
- `*files*.json` → test files/dirs to **delete** from the generated workspace before grading (prevents the LLM from cheating by including the test cases)
- `*.md` → the `start.md` spec handed to the agent
- `*.tar` (optional) → preloaded base image

Results are pushed into the module-level `test_data_list`, which `openhands_app.py` later reads.

**2. Generation** (`openhands/openhands_app.py`)
For each `(model, proName)` pair, `start_openhands` builds an `AppData`, assigning a unique host port starting at 3000. `start_app` then:
- Creates `workspaces/<proName>_bo1/workspace/` and copies `start.md` into it.
- Renders `template/config.template.toml` → `workspaces/<proName>_bo1/config.toml`, substituting `{{VOLUMES}}` (host workspace mount) and `{{MODULE_CONFIG}}` (an `[llm.<sanitized_model_name>]` section built from `moduleName`/`baseUrl`/`sk`).
- Launches an OpenHands container with `AGENT_LLM_CONFIG` pointing at that section and the prompt: *"According to the start.md in the workspace, implement the entire project..."*.
- Polls `container.reload()` every 5 s until the auto-removed container disappears — there is no real exit-code capture, success is assumed if the container vanished cleanly.

The OpenHands container itself mounts `/var/run/docker.sock`, so the agent's nested runtime container is a sibling on the host's Docker daemon, not a child.

**3. Grading** (`openhands/post_processor.py`)
After the agent finishes, `post_process_task`:
- Zips the workspace as an artifact (`workspaces/<id>.zip`).
- Strips package files (`setup.py`, `pyproject.toml`, `requirements*.txt`, etc.) from the workspace.
- Deletes everything listed in `pyTestFileList` so the agent's own copies of test files cannot be reused.
- Writes a `Dockerfile` next to the workspace that `FROM`s the task's reference image and `COPY`s the workspace into `/workspace`.
- Builds `python-test-<task_uuid>` and runs each command in `testShell` inside it.
- Scrapes pytest output with regex (`(\d+) passed`, `(\d+) failed`, `(\d+) error`) and divides `passed` by the txt-file's expected total to compute `success_rate`.

Per-task JSON results are written to `result/<task_uuid>.json`; a dual-output log lands at `workspaces/<id>/log.log`.

## Important Cross-File Coupling

- **`test_data_list` is a module-level global.** `main.py` calls `test_data_service.read_all_test_data()` before `start_openhands(...)`. `only_test.py` does the same. Do not import `start_app` without first populating that list.
- **Port assignment is positional.** `start_openhands` assigns ports as `3000 + index`. If `proNameList` is long enough to overlap with existing services or another concurrent run, containers will fail to bind. The 30-thread `max_pool_size` default in `config.json` therefore implicitly caps to ports 3000–3029.
- **Runtime image version is hardcoded in two places.** `openhands/openhands_app.py:55` references `runtime:0.49-nikolaik` (legacy `create_openhands_container`, unused by the main flow) while line 173 uses `runtime:0.56-nikolaik`. The `readme.md` notes line 176 as the customization point.
- **Image tag convention.** `post_processor.py` hardcodes `{"full_tag": test_data.proName + ":1.0"}` (line 515) and builds `FROM ghcr.io/multimodal-art-projection/nl2repobench/<proName>:1.0`. Adding a new task requires publishing a matching image under that namespace, or uncommenting the `load_docker_image(test_data.imageTar, ...)` line above it and supplying a `.tar`.
- **The agent prompt is hardcoded** in `openhands/openhands_app.py:200`. Changing how tasks are described to the model means editing that string.

## Output Layout

- `workspaces/<proName>_bo1/workspace/` — the agent's working directory (mounted into the runtime)
- `workspaces/<proName>_bo1/config.toml` — rendered OpenHands config for that run
- `workspaces/<proName>_bo1/log.log` — post-processing dual logger output
- `workspaces/<proName>_bo1.zip` — archived workspace snapshot
- `result/<proName>_bo1.json` — aggregated per-task result (status, score, pytest breakdown, paths)
- `logs/application.log` — rotating harness log (10 MB × 5 backups, configured in `logging_config.py`)
