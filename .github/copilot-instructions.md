# LM Studio Python SDK - AI Coding Agent Instructions

## Project Overview

**lmstudio-python** is the official Python SDK for LM Studio, providing both synchronous and asynchronous APIs to interact with LM Studio's native WebSocket API (not the OpenAI compatibility layer). The SDK communicates with a locally running LM Studio desktop application.

### Architecture

- **Dual API Surface**: Fully parallel sync (`sync_api.py`) and async (`async_api.py`) implementations
  - Sync API uses background thread with async WebSocket (`_ws_thread.py`, `_ws_impl.py`)
  - Both share common protocol logic in `json_api.py`
- **Convenience Layer**: Top-level `import lmstudio as lms` provides implicit default `Client` for interactive use
- **Schema Generation**: Python data models are auto-generated from TypeScript/Zod schemas in `lmstudio-js` submodule
- **Plugin System**: Extensible plugin architecture in `src/lmstudio/plugin/` for custom tools and prompt modifications

### Key Components

```
src/lmstudio/
├── sync_api.py          # Synchronous client (Client class at line 1540)
├── async_api.py         # Asynchronous client (AsyncClient)
├── json_api.py          # Shared protocol implementation
├── history.py           # Chat history management
├── schemas.py           # Base schema infrastructure
├── _sdk_models/         # Auto-generated from lmstudio-js JSON schemas
└── plugin/              # Plugin development framework
```

## Development Workflow

### Testing Requirements

**Critical**: Tests require LM Studio desktop app running locally with:
- API server enabled on port 1234
- Specific models loaded (see `tests/README.md`):
  - `text-embedding-nomic-embed-text-v1.5` (embedding)
  - `llama-3.2-1b-instruct` (text LLM)
  - `ZiangWu/MobileVLM_V2-1.7B-GGUF` (visual LLM)
  - `qwen2.5-7b-instruct-1m` (tool-using LLM)

```powershell
# Load test models (unloads any existing instances first)
tox -m load-test-models

# Run tests
tox -m check          # Recommended: runs static checks + tests
tox -m test           # Just run tests (py3.12 by default)
tox -m test_all       # Test all Python versions (3.10-3.14)
```

### Code Quality Commands

```powershell
tox -e format         # Auto-format with ruff
tox -m static         # Run lint + typecheck
tox -e lint           # Ruff linting only
tox -e typecheck      # mypy strict mode checking
```

### Schema Synchronization

When `lmstudio-js` schemas change, regenerate Python models:

```powershell
# Requires lmstudio-js submodule initialized
git submodule update --init --recursive

# Regenerate Python data models from TypeScript schemas
tox -e sync-sdk-schema
```

This runs `sdk-schema/sync-sdk-schema.py` which:
1. Exports JSON schemas from `lmstudio-js` TypeScript/Zod definitions
2. Uses `datamodel-code-generator` to create Python `msgspec` classes
3. Writes to `src/lmstudio/_sdk_models/`

## Code Patterns & Conventions

### Client Lifecycle

```python
# Convenience API (implicit default client)
import lmstudio as lms
model = lms.llm()  # Auto-connects on first use

# Explicit client management (preferred for production)
from lmstudio.sync_api import Client
client = Client()
with client.llm as session:
    model = session.model()
```

### Dual API Consistency

**Always maintain parallel sync/async implementations**. Pattern:
- Sync methods in `sync_api.py` → blocking, uses background thread
- Async methods in `async_api.py` → `async def`, returns awaitables
- Shared logic in `json_api.py` → I/O-agnostic protocol handling

### Error Handling

Use SDK-specific exceptions from `sdk_api.py`:
- `LMStudioClientError` - Client-side issues (connection, configuration)
- `LMStudioPredictionError` - Model prediction failures
- `LMStudioTimeoutError` - Operation timeouts
- `LMStudioCancelledError` - Cancelled operations

Wrap public API methods with `@sdk_public_api()` decorator to filter tracebacks.

### Type Hints

- **Strict mypy**: All code must pass `mypy --strict`
- Use `msgspec.Struct` for data models (not Pydantic, except for testing structured responses)
- Generic types extensively used: see `TypeVar` definitions in API files
- Pattern matching requires Python 3.10+ (use `match`/`case` for type discrimination)

### Logging

Use structured logging via `_logging.py`:
```python
from ._logging import new_logger
logger = new_logger(__name__)
logger.debug("Message", json=data_dict)  # Structured context
```

## Testing Patterns

### Fixtures (conftest.py)

Tests use `pytest` with custom fixtures for LM Studio connections. The test suite is semi-automated requiring manual model loading.

### Test Organization

```
tests/
├── sync/           # Synchronous API tests
├── async/          # Asynchronous API tests
├── support/        # Test utilities and fixtures
└── invalid_plugins/ # Plugin error case tests
```

Run specific test groups:
```powershell
tox -- tests/sync/               # Only sync tests
tox -- tests/test_inference.py   # Specific test file
tox -- -k "test_embedding"       # Tests matching pattern
```

## Common Tasks

### Adding a New API Method

1. Add to `json_api.py` if protocol-level (shared sync/async)
2. Add parallel implementations to `sync_api.py` and `async_api.py`
3. Decorate with `@sdk_public_api()` or `@sdk_public_api_async()`
4. Update type hints and ensure mypy passes
5. Add tests to both `tests/sync/` and `tests/async/`

### Updating Generated Models

When `lmstudio-js` schema changes:
1. Update submodule: `git submodule update --remote`
2. Regenerate: `tox -e sync-sdk-schema`
3. Fix any breaking changes in `json_api.py`, `sync_api.py`, `async_api.py`
4. Run full test suite: `tox -m test_all`

### Plugin Development

See `examples/plugins/` for reference implementations. Plugins use manifest-driven configuration with hook system defined in `src/lmstudio/plugin/hooks/`.

## Critical Notes

- **WebSocket Implementation**: Uses `httpx-ws` with multiplexed channels over single connection
- **Thread Safety**: Sync API uses thread-safe queues and futures for cross-thread communication
- **No OpenAI Compatibility**: This SDK talks to LM Studio's native API, NOT the OpenAI-compatible endpoint
- **Model Identifiers**: Can be model keys (download names) or instance identifiers (loaded instances)
- **Versioning**: Uses semantic-like versioning (X.Y.Z) but with specific policies (see README)

## References

- Main docs: https://lmstudio.ai/docs/python
- Related repo: https://github.com/lmstudio-ai/lmstudio-js (TypeScript SDK, schema source)
- Discord: https://discord.gg/pwQWNhmQTY (#dev-chat channel)
