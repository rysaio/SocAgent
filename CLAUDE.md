# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

SecOps Agent — a security operations AI agent built on **AgentScope v2**. Receives events from external security devices, autonomously investigates via ReAct reasoning, executes or recommends response actions under human supervision.

Official AgentScope docs: https://docs.agentscope.io/

When working with AgentScope APIs, features, or configuration, always consult the official documentation site first before making assumptions.

## Tooling

- **Package manager**: uv (pip replacement)
- **Python**: 3.11
- **Virtualenv**: `.venv/` at project root, managed by uv

## Commands

```bash
# Run the app
uv run python main.py

# Add a dependency
uv add <package>

# Run tests (once test infrastructure exists)
uv run pytest tests/ -v
```

## Environment

`DASHSCOPE_API_KEY` is the only mandatory variable for AgentScope. `REDIS_URL` and `DATABASE_URL` are needed for full stack deployment.
