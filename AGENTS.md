# Agent Guidelines

## Python Environment & Tooling
- This repository manages Python dependencies and virtual environments using **`uv`**.
- **Always run Python scripts and commands via `uv run`**. Do not call bare `python` or `pytest`.
  - Sync dependencies: `uv sync`
  - Start server: `uv run uvicorn main:app --reload`
  - Run worker: `uv run python main.py worker ...`
  - Run tests: `uv run pytest -q`
