# LM Studio Python SDK - AI Coding Agent Instructions

## Project Overview

Official Python SDK for LM Studio, providing sync/async APIs to LM Studio's native WebSocket API (not OpenAI compatibility). Communicates with locally running LM Studio desktop app.

## Architecture

- Dual API: sync (`sync_api.py`) and async (`async_api.py`) implementations sharing `json_api.py` protocol logic.
- Convenience API: `import lmstudio as lms` for implicit default Client.
- Schema generation: Auto-generated from `lmstudio-js` submodule using `msgspec`.
- Plugin system: Extensible in `src/lmstudio/plugin/`.

## Key Components

```
src/lmstudio/
├── sync_api.py (Client at line 1540)
├── async_api.py (AsyncClient)
├── json_api.py (shared protocol)
├── _sdk_models/ (auto-generated)
└── plugin/ (framework)
```

## Development Workflow

### Testing

Requires LM Studio running locally on port 1234 with specific models loaded (see `tests/README.md`).

Steps:
1. Start LM Studio desktop app and enable API server on port 1234.
2. Load required models: `text-embedding-nomic-embed-text-v1.5`, `llama-3.2-1b-instruct`, etc.
3. Run: `tox -m load-test-models` (unloads existing, loads test models).
4. Test: `tox -m check` (static checks + tests) or `tox -m test_all` (all Python versions).

### Code Quality

```bash
tox -e format           # Ruff format
tox -m static           # Lint + typecheck
```

### Schema Sync

When `lmstudio-js` schemas change:
1. Update submodule: `git submodule update --init --recursive`
2. Regenerate models: `tox -e sync-sdk-schema`
3. Fix breaks in API files, run tests.

## Code Patterns

### Client Usage

```python
import lmstudio as lms
model = lms.llm()  # Convenience

# Production
from lmstudio.sync_api import Client
client = Client()
with client.llm as session:
    model = session.model()
```

### Dual API Maintenance

Maintain parallel sync/async methods. Shared logic in `json_api.py`.

Example: Adding a new method `get_status()`:
- In `json_api.py`: Add protocol method.
- In `sync_api.py`: `def get_status(self) -> Status: ...`
- In `async_api.py`: `async def get_status(self) -> Status: ...`

### Error Handling

Use exceptions from `sdk_api.py`: `LMStudioClientError`, `LMStudioPredictionError`, etc. Decorate public APIs with `@sdk_public_api()`.

Example:
```python
@sdk_public_api()
def complete(self, prompt: str) -> str:
    try:
        return self._complete(prompt)
    except Exception as e:
        raise LMStudioPredictionError(f"Completion failed: {e}")
```

### Type Hints

Strict mypy. Use `msgspec.Struct` for models. Generics and pattern matching (Python 3.10+).

Example:
```python
from msgspec import Struct

class ModelInfo(Struct):
    name: str
    size: int

def get_model_info(model_id: str) -> ModelInfo:
    # Implementation
```

### Logging

```python
from ._logging import new_logger
logger = new_logger(__name__)
logger.debug("Message", json=data)
```

## Testing

Semi-automated: requires LM Studio running. Fixtures in `conftest.py`. Test groups in `tests/sync/`, `tests/async/`.

Run specific: `tox -- tests/sync/test_llm_sync.py`

## Common Tasks

- Adding API method: Add to `json_api.py`, then parallel in sync/async, decorate, test both.
- Updating models: Sync submodule, run `tox -e sync-sdk-schema`, fix breaks, test.
- Plugins: See `examples/plugins/`.

## Plugin Development

Plugins extend LM Studio with custom tools or prompt modifications. Framework in `src/lmstudio/plugin/`.

Structure:
- `manifest.json`: Defines plugin metadata and hooks.
- `src/`: Implementation code.

Example manifest (from `examples/plugins/dice-tool/`):
```json
{
  "name": "dice-tool",
  "description": "A tool for rolling dice",
  "hooks": ["tool"]
}
```

Hook implementation:
```python
from lmstudio.plugin import ToolHook

class DiceTool(ToolHook):
    def get_tools(self):
        return [{
            "name": "roll_dice",
            "description": "Roll a die with specified sides",
            "parameters": {
                "type": "object",
                "properties": {
                    "sides": {"type": "integer", "default": 6}
                }
            }
        }]

    def call_tool(self, name, arguments):
        if name == "roll_dice":
            import random
            sides = arguments.get("sides", 6)
            return random.randint(1, sides)
```

Load plugin: Use `lmstudio plugin load <path>` or via API.
