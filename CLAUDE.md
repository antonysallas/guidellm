# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GuideLLM is a platform for evaluating and optimizing the deployment of large language models (LLMs). It simulates real-world inference workloads to assess performance, resource requirements, and cost implications of deploying LLMs on various hardware configurations.

This is a **dual-language project**:
- **Python backend** (`src/guidellm/`): Core benchmarking engine with CLI
- **TypeScript/Next.js frontend** (`src/ui/`): Web UI for visualizing benchmark results

## Common Development Commands

### Python Backend

**Installation:**
```bash
pip install -e ./[dev]
```

**Running Tests:**
```bash
tox                    # Run all tests
tox -e test-unit       # Unit tests only
tox -e test-integration # Integration tests only
tox -e test-e2e        # End-to-end tests only
```

**Code Quality:**
```bash
tox -e quality         # Check code quality (linting, formatting)
tox -e style           # Auto-fix style issues
tox -e types           # Run type checking with mypy
tox -e links           # Check markdown links
```

**Running a Single Test:**
```bash
pytest tests/unit/path/to/test_file.py::test_function_name
```

**Building:**
```bash
tox -e build
```

**Cleaning:**
```bash
tox -e clean
```

### TypeScript/Next.js UI

**Installation:**
```bash
npm install
```

**Development:**
```bash
npm run dev            # Start dev server at http://localhost:3000
npm run build          # Build for production (outputs to src/ui/out)
npm run serve          # Serve the built static site
```

**Testing:**
```bash
npm run test           # Run all UI tests
npm run test:unit      # Unit tests only
npm run test:integration # Integration tests only
npm run test:e2e       # End-to-end tests (Cypress)
npm run coverage       # Run tests with coverage
npm run test:watch     # Run tests in watch mode
```

**Code Quality:**
```bash
npm run lint           # Run ESLint
npm run format         # Format code with Prettier
npm run type-check     # TypeScript type checking
```

## Architecture

GuideLLM follows a modular pipeline architecture:

```
DatasetCreator → RequestLoader → Scheduler → RequestsWorker(s) → Backend
                                                ↓
                                    BenchmarkAggregator → Benchmarker
```

### Core Components

1. **Backend** (`src/guidellm/backend/`): Abstract interface for generative AI backends (OpenAI-compatible HTTP servers like vLLM). Processes requests and generates responses.

2. **DatasetCreator** (`src/guidellm/dataset/`): Loads data from HuggingFace datasets, local files (CSV, JSON, JSONL), or generates synthetic data. Converts data into standardized format.

3. **RequestLoader** (`src/guidellm/request/`): Converts dataset items into backend-compatible requests for text or chat completions.

4. **Scheduler** (`src/guidellm/scheduler/`): Manages request scheduling using multiprocessing and asyncio. Handles different rate strategies (synchronous, throughput, concurrent, constant, poisson, sweep).

5. **RequestsWorker** (`src/guidellm/scheduler/worker.py`): Worker process that pulls requests from queue, sends to backend, and returns results.

6. **Benchmarker** (`src/guidellm/benchmark/benchmarker.py`): Orchestrates multiple benchmark runs, manages progress tracking, and compiles final reports.

7. **BenchmarkAggregator** (`src/guidellm/benchmark/aggregator.py`): Collects and aggregates results from benchmark runs, computes statistics.

### Key Modules

- **Scenarios** (`src/guidellm/benchmark/scenario.py`): Configuration objects that define benchmark parameters. Supports loading from YAML files or built-in presets.

- **Config** (`src/guidellm/config.py`): Environment variable configuration using Pydantic settings with `GUIDELLM__*` prefix.

- **Presentation** (`src/guidellm/presentation/`): Handles output formatting and HTML report generation. The `injector.py` embeds benchmark data into HTML templates.

- **UI** (`src/ui/`): Next.js static site that visualizes benchmark results. Uses Redux for state management and Material-UI/Nivo for components/charts.

## CLI Structure

The main CLI is defined in `src/guidellm/__main__.py`:

- `guidellm benchmark run`: Run new benchmarks
- `guidellm benchmark from-file`: Load and re-export saved benchmarks
- `guidellm preprocess dataset`: Convert datasets to specific token sizes
- `guidellm config`: Print environment variable configuration

All CLI options can be set via environment variables with `GUIDELLM__*` prefix (e.g., `GUIDELLM_TARGET`, `GUIDELLM_RATE_TYPE`).

## Testing Strategy

- **Unit tests** (`tests/unit/`): Test individual components with mocking
- **Integration tests** (`tests/integration/`): Test component interactions without mocking
- **E2E tests** (`tests/e2e/`): Test entire system and user workflows
- **UI tests** (`tests/ui/`): Jest for unit/integration, Cypress for E2E

Use pytest markers for test categorization: `@pytest.mark.smoke`, `@pytest.mark.sanity`, `@pytest.mark.regression`

## Code Quality Standards

- **Python**: Uses ruff for linting/formatting, mypy for type checking, mdformat for markdown
- **TypeScript**: Uses ESLint, Prettier, and TypeScript compiler
- **Pre-commit hooks**: Available via `pre-commit install` (runs quality checks on commit)

## Configuration & Logging

Configuration is managed via Pydantic Settings with hierarchical loading:
1. Default values
2. Environment variables (`GUIDELLM__*`)
3. CLI arguments (highest priority)

Logging configuration:
- `GUIDELLM__LOGGING__DISABLED`: Disable logging (default: false)
- `GUIDELLM__LOGGING__CONSOLE_LOG_LEVEL`: Console log level (DEBUG, INFO, WARNING, ERROR, CRITICAL)
- `GUIDELLM__LOGGING__LOG_FILE`: Log file path (default: guidellm.log if level set)
- `GUIDELLM__LOGGING__LOG_FILE_LEVEL`: File log level (default: INFO)

Use `guidellm config` to validate environment variable settings.

## Output Formats

GuideLLM supports multiple output formats (detected by file extension):
- **JSON** (`.json`): Full benchmark data with all statistics and request details
- **YAML** (`.yaml`): Human-readable alternative to JSON
- **CSV** (`.csv`): Tabular format for spreadsheet analysis
- **HTML** (`.html`): Interactive visualization using the Next.js UI

Results include:
1. **Benchmarks Metadata**: Configuration and parameters used
2. **Benchmarks Info**: High-level summary of each run
3. **Benchmarks Stats**: Detailed metrics (request rate, latency, TTFT, ITL, etc.)

## Important Implementation Notes

- The scheduler uses both multiprocessing (for workers) and asyncio (for concurrent requests) to maximize throughput
- Synthetic data generation uses statistical distributions (normal, uniform) to create realistic token patterns
- The UI is statically generated and can be embedded with benchmark data for self-contained HTML reports
- Backend validation happens at startup to ensure connectivity and model availability
- Warmup/cooldown phases can be configured to exclude initialization and teardown from metrics
- Request sampling strategies affect dataset iteration (random shuffle vs. sequential)

## Development Workflow

1. Make changes in feature branch
2. Run quality checks: `tox -e quality` (Python) or `npm run lint && npm run type-check` (UI)
3. Run tests: `tox -e test-unit` or `npm run test`
4. Fix style: `tox -e style` (Python) or `npm run format` (UI)
5. Verify types: `tox -e types` (Python) or `npm run type-check` (UI)
6. Run full test suite: `tox` or `npm run coverage`
7. Open pull request with summary, details, test plan, and related issues

## Container Support

Container images published at `ghcr.io/vllm-project/guidellm`:
- `latest`, `stable`: Latest tagged release
- `nightly`: Built from main branch daily
- `v*.*.*`: Specific version tags
- `pr-*`: Development builds (do not use in production)

Container runs `guidellm benchmark run` by default. CLI options can be passed as environment variables (`GUIDELLM_*`).
