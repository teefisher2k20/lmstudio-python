"""
LM Studio Python SDK - Quick Start Examples

This module demonstrates three ways to use the lmstudio-python SDK:
1. Interactive Convenience API (default client)
2. Synchronous Scoped Resource API (context managers)
3. Asynchronous Structured Concurrency API (async/await)
"""

import lmstudio as lms
import asyncio


# =============================================================================
# Example 1: Interactive Convenience API
# =============================================================================
# Best for: Interactive prompts, Jupyter notebooks, quick scripts
def example_convenience_api():
    """Simple, convenient API using the default client instance."""
    model = lms.llm("qwen/qwen3-4b-2507")
    result = model.respond("What is the meaning of life?")
    print(result)


# =============================================================================
# Example 2: Synchronous Scoped Resource API
# =============================================================================
# Best for: Production code requiring deterministic resource cleanup
def example_scoped_resource_api():
    """
    Scoped resource management with context managers.
    Resources are released deterministically when exiting the context.
    """
    with lms.Client() as client:
        model = client.llm.model("qwen/qwen3-4b-2507")
        result = model.respond("What is the meaning of life?")
        print(result)


# =============================================================================
# Example 3: Asynchronous Structured Concurrency API
# =============================================================================
# Best for: Async applications following structured concurrency principles
# Requires: Python SDK version 1.5.0 or later
async def example_async_api():
    """
    Asynchronous API for concurrent operations.
    Uses structured concurrency to manage background tasks properly.
    """
    async with lms.AsyncClient() as client:
        model = await client.llm.model("qwen/qwen3-4b-2507")
        result = await model.respond("What is the meaning of life?")
        print(result)


# =============================================================================
# Timeout Configuration (SDK 1.5.0+)
# =============================================================================
def configure_sync_timeout():
    """Configure timeout for synchronous API (default: 60 seconds)."""
    # Get current timeout
    current_timeout = lms.get_sync_api_timeout()
    print(f"Current timeout: {current_timeout} seconds")
    
    # Set custom timeout (in seconds)
    lms.set_sync_api_timeout(120)
    
    # Disable timeout entirely
    lms.set_sync_api_timeout(None)


async def example_async_timeout():
    """
    Async API uses standard async timeout mechanisms.
    No built-in timeout - use asyncio.wait_for() or similar.
    """
    async with lms.AsyncClient() as client:
        model = await client.llm.model("qwen/qwen3-4b-2507")
        
        # Apply timeout using asyncio
        try:
            result = await asyncio.wait_for(
                model.respond("What is the meaning of life?"),
                timeout=30.0  # 30 second timeout
            )
            print(result)
        except asyncio.TimeoutError:
            print("Request timed out after 30 seconds")


# =============================================================================
# Main execution
# =============================================================================
if __name__ == "__main__":
    print("Example 1: Convenience API")
    example_convenience_api()
    
    print("\nExample 2: Scoped Resource API")
    example_scoped_resource_api()
    
    print("\nExample 3: Asynchronous API")
    asyncio.run(example_async_api())
    
    print("\nTimeout Configuration")
    configure_sync_timeout()
    asyncio.run(example_async_timeout())