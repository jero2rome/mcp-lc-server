Purpose

You (the coding agent) are collaborating on a Model Context Protocol (MCP) server for Liquid Chromatography (LC).
Your job: add/modify tools, schemas, and adapters so LLM-based IDE agents can orchestrate LC workflows via MCP.

Quick start
•Python: 3.11+
•Package manager: prefer uv (fallback: pip)
•Install (dev):

uv venv && uv pip install -e ".[dev]"
# or: python -m venv .venv && source .venv/bin/activate && pip install -e ".[dev]"


•Run the MCP server (dev):

uv run mcp-lc-server  # if a console_script is defined
# or:
python -m mcp_lc_server


•Health check: server prints a list of tools on startup; exit with Ctrl+C.

What this project does
•Exposes LC capabilities to any MCP-aware client (Claude Code, Cursor, etc.) via MCP tools: request/response JSON RPC.
•Adapters can bridge to real instruments or to a simulator (for CI and local dev).

Repo layout (expected)

mcp_lc_server/
  __init__.py
  server.py            # MCP server bootstrap & tool registration
  tools/
    methods.py         # query/create LC methods
    runs.py            # start/check/cancel runs
    instruments.py     # discovery + status
    analysis.py        # basic chromatogram operations (if simulated)
  schemas.py           # Pydantic models for tool I/O
  adapters/
    orchestrator.py    # calls your orchestration API (HTTP/Dapr/Kafka)
    simulator.py       # local fake implementation for CI
pyproject.toml
README.md
AGENTS.md
tests/

If file paths differ, stick to the intent above.

Environment

Set these (or use sensible fallbacks in code):
•LC_ORCH_URL – base URL of orchestration API (e.g., Dapr sidecar or gateway)
•LC_AUTH_TOKEN – bearer token for secured endpoints (optional)
•LC_SIM_MODE – "true" to force simulator (for CI/local)
•LC_BROKER_URL – Kafka/Redis/… if you publish/subscribe events (optional)

How to use in MCP clients
•Claude / Cursor, etc.: add a client entry that points to the server command:

{
  "mcpServers": {
    "lc": {
      "command": "mcp-lc-server",
      "args": [],
      "env": { "LC_SIM_MODE": "true" }
    }
  }
}



(Exact location depends on the client; add notes in README for user setup.)

Tools (contract you must maintain)

Keep tool names stable; evolve via new optional fields. Add examples in docstrings.

lc.list_instruments() -> Instrument[]
•Desc: List available LC instruments and connection status.
•Returns: [{ id, name, vendor, model, status }]

lc.get_methods(query?) -> Method[]
•Desc: Search or list known LC methods.
•Args: query (string, optional)
•Returns: method metadata + key parameters (columns, mobile phases, gradient, temp, flow).

lc.create_method(MethodSpec) -> Method
•Desc: Create or register a method (sim or real).
•Validates: gradient times monotonic, %B in [0,100], pressure limits, etc.

lc.start_run(RunRequest) -> RunHandle
•Desc: Start a run on an instrument with a method + sample info.
•Args: { instrument_id, method_id|method_spec, vial, sample_id, replicate? }
•Returns: { run_id, instrument_id, status, eta? }

lc.get_run_status(run_id) -> RunStatus
•Desc: Poll status.
•Returns: { state: "queued|running|done|error|canceled", progress?, message? }

lc.get_result(run_id) -> ResultBundle
•Desc: Retrieve chromatogram + basic metrics (Rt, peak areas, Rs if available).

lc.cancel_run(run_id) -> Ack

Include JSON examples in each tool’s docstring so agents can learn I/O shapes.

Schemas (Pydantic types)
•Instrument { id:str, name:str, vendor:str, model:str, status:"online|offline|busy" }
•Method { id, name, column, mobile_phase_a, mobile_phase_b, gradient:[{time,min,percent_b}], flow_ml_min, temp_c, wavelength_nm? }
•RunRequest { instrument_id, method_id|method_spec, vial, sample_id, replicate? }
•RunStatus { state, progress?, message? }
•ResultBundle { peaks:[{rt, area, width, height}], chromatogram:base64_png|csv, system_pressure?:[...]}
•Prefer snake_case and explicit units.

Dev tasks you (agent) are allowed to do
•Implement/extend tools above with input validation and typed schemas.
•Add a simulator that returns deterministic chromatograms for fixed method seeds.
•Wire an adapter to the orchestration API (HTTP client with retries & timeouts).
•Add tests (pytest) with fixtures covering:
•happy path (simulated run),
•invalid gradients / out-of-range parameters,
•network failures (adapter) with retry/backoff.
•Add ruff + mypy; keep CI green.

Commands & checks

# lint & type
uvx ruff check .
uvx mypy mcp_lc_server

# tests
uvx pytest -q

# format
uvx ruff format .

Code style & PR rules
•Keep functions small; no side effects in model/serializer code.
•Don’t break tool names or response shapes; add new fields with defaults.
•Every new tool needs:
1.Docstring with example request/response,
2.Unit tests,
3.Entry in this AGENTS.md.
•Security: never log secrets; pass tokens via headers; sanitize file paths.
•Do not alter CI, release tags, or license files unless the task requires it.

Release (when packaging)
•Bump version in pyproject.toml.
•uv build → produce wheel/sdist.
•Tag as vX.Y.Z, update CHANGELOG.

What good contributions look like
•Adds feature + tests + docs in one PR.
•Keeps simulator parity with real adapter (same tool contract).
•Clear error messages; predictable exceptions.

Open questions for future work
•Peak detection settings (S/N thresholds) in simulator.
•Result normalization across vendors.
•Async streaming updates via MCP notifications (optional).

⸻

Why this file exists

Codex (local or cloud) and other coding agents look for AGENTS.md to understand how to build, run, test, and change your repo—much like a README for agents. GitHub’s Copilot coding agent also supports AGENTS.md for custom instructions.

⸻

