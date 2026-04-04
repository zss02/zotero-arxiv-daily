# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Zotero-arXiv-Daily recommends new arXiv/bioRxiv/medRxiv papers based on a user's Zotero library. It computes embedding similarity between new papers and the user's existing library, generates TLDRs via LLM, and delivers results by email. Designed to run as a GitHub Actions workflow at zero cost.

## Commands

```bash
# Run the application
uv run src/zotero_arxiv_daily/main.py

# Run tests (excludes CI-only tests by default)
uv run pytest

# Run all tests including CI-marked ones (needs mock SMTP + mock OpenAI services)
uv run pytest -m ""

# Run a single test
uv run pytest tests/test_glob_match.py::test_glob_match -v

# Install/sync dependencies
uv sync
```

No linter or formatter is configured.

## Architecture

The app follows a linear pipeline orchestrated by `Executor` (`src/zotero_arxiv_daily/executor.py`):

1. **Fetch Zotero corpus** — retrieves user's library papers via pyzotero API
2. **Filter corpus** — applies `include_path` glob patterns to select relevant collections
3. **Retrieve new papers** — fetches from configured sources (arXiv RSS, bioRxiv/medRxiv REST API)
4. **Rerank** — scores candidates by weighted similarity to corpus (newer Zotero papers weighted higher)
5. **Generate TLDRs + affiliations** — via OpenAI-compatible LLM API
6. **Render + send email** — HTML email via SMTP

### Plugin Systems

**Retrievers** (`src/zotero_arxiv_daily/retriever/`): Register via `@register_retriever` decorator, discovered by `get_retriever_cls()`. Each retriever implements `_retrieve_raw_papers()` and `convert_to_paper()`.

**Rerankers** (`src/zotero_arxiv_daily/reranker/`): Register via `@register_reranker` decorator, discovered by `get_reranker_cls()`. Two implementations: `local` (sentence-transformers) and `api` (OpenAI-compatible embeddings endpoint).

### Configuration

Uses Hydra + OmegaConf. Config is composed from `config/base.yaml` (defaults) + `config/custom.yaml` (user overrides). Environment variables are interpolated via `${oc.env:VAR_NAME,default}` syntax. Entry point uses `@hydra.main`.

### Data Classes

`Paper` and `CorpusPaper` in `src/zotero_arxiv_daily/protocol.py`. `Paper` has LLM-powered methods (`generate_tldr`, `generate_affiliations`) that call the OpenAI API directly.

## Testing

Tests marked `@pytest.mark.ci` require external services (mock SMTP via MailHog on port 1025, mock OpenAI server on port 30000) and are skipped locally by default (`addopts = "-m 'not ci'"` in pyproject.toml). CI configures these as service containers.

## Git Workflow

- PRs should target the `dev` branch, not `main`
- Current development branch: `dev`
