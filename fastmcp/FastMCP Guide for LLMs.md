# Ultimate Python Model Context Protocol (MCP) Guide

## Overview

The Model Context Protocol (MCP) provides a standardized interface for LLM applications to interact with external data and functionality. FastMCP is the Python implementation of this protocol, offering a high-level, intuitive interface for developers.

## Installation and Setup

```bash
# Using uv (recommended)
uv add "mcp[cli]"

# Using pip
pip install mcp

# Optional: Install with all extras
pip install "mcp[cli,dev,test]"
```
> ⚠️ **Python Version Compatibility**: FastMCP works best with Python 3.9+ but may have issues with Python 3.13+. For best results, use Python 3.11.

## Core Components

### Server Creation

```python
from mcp.server.fastmcp import FastMCP

# Basic server
server = FastMCP("MyApp")

# Server with dependencies
server = FastMCP(
    "AnalyticsApp",
    dependencies=[
        "pydantic>=2.0",
        "pandas",
        "numpy",
        "scikit-learn",
        "asyncpg",
    ]
)
```

### Type System and Validation

FastMCP uses Pydantic for type validation and schema generation:

```python
from typing import Annotated
from pydantic import BaseModel, Field

# Complex type definition
class UserProfile(BaseModel):
    name: str = Field(..., description="User's full name")
    age: Annotated[int, Field(ge=0, lt=150)]
    email: Annotated[str, Field(pattern=r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")]
    preferences: dict[str, bool] = Field(
        default_factory=dict,
        description="User preferences"
    )

# Tool using complex type
@server.tool()
def update_profile(
    profile: UserProfile,
    notify: bool = Field(
        default=True,
        description="Send notification email"
    )
) -> dict:
    """Update a user's profile
    
    Args:
        profile: Complete user profile data
        notify: Whether to send notification
    
    Returns:
        Updated profile data
    """
    return profile.model_dump()
```

## Tools

Tools are functions that can perform actions or computations. They support both synchronous and asynchronous operations.

### Basic Tools

```python
@server.tool()
def calculate(
    operation: Annotated[str, Field(
        description="Operation to perform",
        pattern="^(add|subtract|multiply|divide)$"
    )],
    a: float = Field(..., description="First number"),
    b: float = Field(..., description="Second number")
) -> float:
    """Perform basic arithmetic
    
    Args:
        operation: Mathematical operation
        a: First operand
        b: Second operand
    
    Returns:
        Result of operation
    
    Raises:
        ValueError: For invalid operations
        ZeroDivisionError: When dividing by zero
    """
    ops = {
        "add": lambda x, y: x + y,
        "subtract": lambda x, y: x - y,
        "multiply": lambda x, y: x * y,
        "divide": lambda x, y: x / y
    }
    
    if operation not in ops:
        raise ValueError(f"Invalid operation: {operation}")
        
    return ops[operation](a, b)
```

### Async Tools with Progress

```python
from mcp.server.fastmcp import Context

@server.tool()
async def process_dataset(
    dataset_uri: str,
    batch_size: int = Field(
        default=1000,
        ge=100,
        le=10000,
        description="Number of records per batch"
    ),
    ctx: Context
) -> dict:
    """Process a large dataset with progress tracking
    
    Args:
        dataset_uri: URI of dataset to process
        batch_size: Records per batch
        ctx: MCP context
    """
    total_records = await count_records(dataset_uri)
    processed = 0
    results = []

    async for batch in load_batches(dataset_uri, batch_size):
        # Process batch
        batch_result = await process_batch(batch)
        results.append(batch_result)
        
        # Update progress
        processed += len(batch)
        await ctx.report_progress(
            current=processed,
            total=total_records,
            message=f"Processed {processed}/{total_records} records"
        )
        
        # Log information
        ctx.info(f"Batch {len(results)} complete")
        
    return {
        "total_processed": processed,
        "batches": len(results),
        "summary": aggregate_results(results)
    }
```

### Tools with Complex Types

```python
class DatabaseConfig(BaseModel):
    host: str = Field(..., description="Database host")
    port: int = Field(default=5432, ge=1, le=65535)
    database: str
    username: str
    password: str = Field(..., exclude=True)  # Sensitive field
    
class QueryConfig(BaseModel):
    table: str
    columns: list[str] = Field(default_factory=list)
    where: dict[str, str] = Field(default_factory=dict)
    limit: int = Field(default=100, ge=1, le=1000)

@server.tool()
async def query_database(
    db_config: DatabaseConfig,
    query_config: QueryConfig,
    ctx: Context
) -> list[dict]:
    """Execute a database query
    
    Args:
        db_config: Database connection details
        query_config: Query parameters
        ctx: MCP context
    
    Returns:
        Query results
    """
    try:
        async with connect_db(db_config) as conn:
            results = await execute_query(conn, query_config)
            return results
    except Exception as e:
        ctx.error(f"Database error: {str(e)}")
        raise
```

## Resources

Resources provide read-only access to data through URI-like patterns.

### Static Resources

```python
@server.resource("config://app")
def get_app_config() -> dict:
    """Get application configuration"""
    return {
        "version": "1.0.0",
        "environment": "production",
        "features": {
            "beta": True,
            "experimental": False
        }
    }
```

### Dynamic Resources

```python
@server.resource("users://{user_id}/{section}")
async def get_user_section(
    user_id: str,
    section: Annotated[str, Field(pattern="^(profile|settings|history)$")]
) -> dict:
    """Get a section of user data
    
    Args:
        user_id: User identifier
        section: Data section to retrieve
    """
    async with get_user_db() as db:
        data = await db.fetch_one(
            "SELECT * FROM user_data WHERE id = $1 AND section = $2",
            user_id,
            section
        )
        return dict(data) if data else None
```

### Resource Collections

```python
@server.resource("reports://monthly/{year}/{month}")
def get_monthly_report(
    year: Annotated[int, Field(ge=2000, le=2100)],
    month: Annotated[int, Field(ge=1, le=12)]
) -> dict:
    """Get monthly report data
    
    Args:
        year: Report year
        month: Report month (1-12)
    """
    return load_report(year, month)
```

## Prompts

Prompts in FastMCP allow you to create structured interactions with LLMs. Here's how to implement them correctly:

```python
@mcp.prompt(
    name="analyze-data",
    description="Analyze the provided dataset and provide insights"
)
def analyze_data(ctx: Context, parameters: dict) -> dict:
    """
    Process data analysis prompt
    
    Args:
        ctx: MCP context
        parameters: Prompt parameters with data and configuration
    
    Returns:
        Dictionary with markdown-formatted analysis
    """
    # Extract parameters with defaults
    data = parameters.get('data', [])
    threshold = parameters.get('threshold', 100)
    
    # Process data
    results = process_data(data, threshold)
    
    return {
        "type": "markdown",
        "content": f"""## Analysis Results

{results}
"""
    }
```

Key points about prompts:
- Keep decorator simple with just `name` and `description`
- Accept `ctx` and `parameters` arguments
- Return a dict with `type` and `content` keys
- Use markdown formatting for structured output
- Handle parameter defaults within the function

Example of a more complex prompt:

```python
@mcp.prompt(
    name="analyze-regions",
    description="Analyze regions for carbon intensity discounts"
)
def analyze_regions(ctx: Context, parameters: dict) -> dict:
    """Process region analysis prompt"""
    # Get parameters with defaults
    threshold = parameters.get('threshold', 100)
    discount = parameters.get('discount', 3)
    
    # Find qualifying regions
    qualifying = [
        region.format_details()
        for region in REGIONS
        if region.carbon_intensity < threshold
    ]

    return {
        "type": "markdown",
        "content": f"""## Qualifying Regions for {discount}% Discount

The following regions have carbon intensity below {threshold} gCO2eq/kWh:

{chr(10).join(f"* {region}" for region in qualifying)}
"""
    }
```

## State and Session Management

### Session Storage

```python
from pathlib import Path
import json

class SessionManager:
    def __init__(self, base_dir: Path):
        self.base_dir = base_dir
        self.base_dir.mkdir(parents=True, exist_ok=True)
    
    def get_session_path(self, session_id: str) -> Path:
        return self.base_dir / f"{session_id}.json"
    
    def load(self, session_id: str) -> dict:
        path = self.get_session_path(session_id)
        if path.exists():
            return json.loads(path.read_text())
        return {}
    
    def save(self, session_id: str, data: dict):
        path = self.get_session_path(session_id)
        path.write_text(json.dumps(data))

@server.tool()
async def stateful_operation(
    data: dict,
    ctx: Context
) -> dict:
    """Operation that maintains state
    
    Args:
        data: New data to process
        ctx: MCP context
    """
    # Get existing state
    state = ctx.session.get("operation_state", {})
    
    # Update state
    state.update(data)
    
    # Store updated state
    ctx.session["operation_state"] = state
    
    return state
```

## Advanced Features

### Custom Transports

```python
from mcp.server.lowlevel import Server
from mcp.server.models import InitializationOptions
import mcp.server.stdio

async def run_custom_server():
    server = Server("custom-server")
    
    # Configure server...
    
    async with mcp.server.stdio.stdio_server() as (read, write):
        await server.run(
            read_stream=read,
            write_stream=write,
            initialization_options=InitializationOptions(
                server_name="custom",
                server_version="1.0.0",
                capabilities=server.get_capabilities()
            )
        )
```

### Error Recovery

```python
@server.tool()
async def resilient_operation(
    retries: int = Field(default=3, ge=1, le=5),
    ctx: Context
) -> dict:
    """Operation with retry logic
    
    Args:
        retries: Number of retry attempts
        ctx: MCP context
    """
    for attempt in range(retries):
        try:
            result = await perform_operation()
            return result
        except TransientError as e:
            ctx.warning(f"Attempt {attempt + 1} failed: {str(e)}")
            if attempt < retries - 1:
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Progress Monitoring

```python
@server.tool()
async def multi_stage_operation(ctx: Context) -> dict:
    """Operation with multiple progress stages"""
    stages = [
        ("Initialization", 10),
        ("Data Processing", 50),
        ("Validation", 20),
        ("Finalization", 20)
    ]
    
    current_progress = 0
    results = {}
    
    for stage_name, stage_weight in stages:
        ctx.info(f"Starting {stage_name}")
        
        # Process stage
        stage_result = await process_stage(stage_name)
        results[stage_name] = stage_result
        
        # Update progress
        current_progress += stage_weight
        await ctx.report_progress(
            current=current_progress,
            total=100,
            message=f"Completed {stage_name}"
        )
    
    return results
```

## Testing

### Command Line Testing

Before deploying your FastMCP server with Claude Desktop or other LLM interfaces, always test it from the command line:

```bash
# Activate your virtualenv
source venv/bin/activate

# Run your server directly
python your_server.py
```

This helps identify any immediate issues with imports, syntax, or dependencies.

### Logging Best Practices

FastMCP manages its own logging. Avoid setting up custom logging to prevent conflicts. The built-in logging will show server initialization and request handling:

```python
# Do NOT add custom logging setup
# Let FastMCP handle logging internally

# Simple server startup
if __name__ == "__main__":
    try:
        mcp.run()
    except Exception as e:
        print(f"Server failed to start: {str(e)}")
        raise
```

### Unit Tests

```python
import pytest
from mcp.server.fastmcp import FastMCP

@pytest.fixture
def server():
    return FastMCP("TestServer")

@pytest.fixture
def complex_server():
    server = FastMCP(
        "ComplexServer",
        dependencies=["pandas", "numpy"]
    )
    
    # Add test tools
    @server.tool()
    def test_tool(x: int) -> int:
        return x * 2
    
    return server

def test_basic_tool(server):
    @server.tool()
    def add(a: int, b: int) -> int:
        return a + b
    
    result = add(2, 3)
    assert result == 5

@pytest.mark.asyncio
async def test_async_tool(server):
    @server.tool()
    async def process(data: str) -> str:
        return f"Processed: {data}"
    
    result = await process("test")
    assert result == "Processed: test"
```

### Integration Tests

```python
@pytest.mark.integration
async def test_database_integration(complex_server):
    @complex_server.tool()
    async def db_operation(query: str) -> list:
        async with get_test_db() as conn:
            return await conn.fetch(query)
    
    result = await db_operation("SELECT * FROM test_table")
    assert len(result) > 0
```

## Security Considerations

1. Input Validation
```python
@server.tool()
def secure_operation(
    filename: Annotated[str, Field(pattern=r'^[\w\-. ]+$')],
    content: str
) -> bool:
    """Securely handle file operations
    
    Args:
        filename: Safe filename (alphanumeric, dots, dashes, spaces)
        content: File content
    """
    safe_path = Path("safe_directory") / filename
    if not safe_path.parent == Path("safe_directory"):
        raise ValueError("Invalid path")
    
    safe_path.write_text(content)
    return True
```

2. Resource Access Control
```python
@server.resource("secure://{path}")
async def get_secure_resource(
    path: str,
    ctx: Context
) -> str:
    """Access controlled resource
    
    Args:
        path: Resource path
        ctx: MCP context
    """
    if not await check_access(ctx.session, path):
        raise PermissionError(f"Access denied to {path}")
    
    return await load_resource(path)
```

## Best Practices

1. Documentation
   - Use detailed docstrings
   - Include type hints
   - Document exceptions and edge cases
   
2. Error Handling
   - Use specific exception types
   - Provide clear error messages
   - Handle cleanup in finally blocks
   
3. Resource Management
   - Use context managers
   - Clean up resources properly
   - Handle concurrent access
   
4. Testing
   - Write comprehensive tests
   - Use fixtures for common setup
   - Test error cases
   
5. Security
   - Validate all inputs
   - Use proper access controls
   - Handle sensitive data carefully

## Common Patterns

1. Resource Hierarchy
```python
@server.resource("api://{version}/{endpoint}/{id}")
def get_api_resource(
    version: Annotated[str, Field(pattern=r"^v\d+$")],
    endpoint: str,
    id: str
) -> dict:
    """Access API resource
    
    Args:
        version: API version (e.g., "v1")
        endpoint: API endpoint
        id: Resource identifier
    """
    return fetch_api_resource(version, endpoint, id)
```

2. Batch Operations
```python
@server.tool()
async def batch_process(
    items: list[dict],
    batch_size: int = Field(default=100, ge=1, le=1000),
    ctx: Context
) -> dict:
    """Process items in batches
    
    Args:
        items: Items to process
        batch_size: Size of each batch
        ctx: MCP

# Handling Async Functions in FastMCP Resources

## Core Concepts

FastMCP resources must return concrete data rather than coroutines or futures. When working with async functions in FastMCP resources, you need to handle three key challenges:

1. Resources cannot be async functions
2. The event loop is managed by FastMCP
3. Async operations must complete before returning

## Solution Pattern

The solution involves creating a thread-based wrapper to run async functions synchronously:

```python
def run_async_in_thread(async_func):
    """Run an async function to completion in a separate thread"""
    def wrapper(*args, **kwargs):
        with concurrent.futures.ThreadPoolExecutor() as pool:
            future = pool.submit(lambda: asyncio.run(async_func(*args, **kwargs)))
            return future.result()
    return wrapper
```

## Common Mistakes to Avoid

1. **Using Async Resources**
   ```python
   # Wrong - returns a coroutine object
   @mcp.resource("docs://example")
   async def example_docs():
       return await perform_search()
   ```

2. **Manual Event Loop Management**
   ```python
   # Wrong - conflicts with FastMCP's event loop
   loop = asyncio.get_event_loop()
   loop.run_until_complete(perform_search())
   ```

3. **Returning Futures**
   ```python
   # Wrong - returns an uncompleted Future
   future = asyncio.Future()
   asyncio.create_task(some_async_func())
   return future
   ```

## Correct Implementation

1. Create async functions for your core operations:
   ```python
   async def perform_search(query: str, limit: int = 5):
       # Async implementation
       pass
   ```

2. Create a synchronous wrapper:
   ```python
   sync_perform_search = run_async_in_thread(perform_search)
   ```

3. Use the synchronous wrapper in resources:
   ```python
   @mcp.resource("docs://example")
   def example_docs():
       return sync_perform_search("query", 5)
   ```

## Best Practices

- Keep async functions for core operations
- Use the thread wrapper pattern for resource implementations
- Avoid direct event loop manipulation
- Return concrete data from resources
- Handle exceptions within async functions

## Technical Details

The solution works by:
1. Creating a new thread for each async operation
2. Running a new event loop in that thread
3. Executing the async function to completion
4. Returning the final result

This approach maintains FastMCP's event loop while ensuring async operations complete properly.

## Troubleshooting Common Issues

### Server Connection Issues

If you encounter "Could not attach server" or similar connection issues:

1. Test from command line first
2. Check Python version compatibility (use 3.9-3.12)
3. Verify imports (especially `Context` from `fastmcp`)
4. Remove any custom logging setup
5. Ensure prompt structure matches examples above

### Logging Issues

If you see logging-related errors:

1. Remove any custom logging configuration
2. Don't create custom loggers for your server
3. Let FastMCP handle all logging internally
4. Use simple print statements for debugging if needed

### Prompt Issues

Common prompt-related problems:

1. Including `template` in decorator (not supported)
2. Missing `parameters` argument in function
3. Wrong return type (must be dict with `type` and `content`)
4. Custom logging in prompt handlers

## Best Practices

1. Development Workflow:
   - Always test from command line first
   - Use Python 3.11 for best compatibility
   - Keep prompt decorators simple
   - Let FastMCP handle logging

2. Code Structure:
   - Import Context from fastmcp directly
   - Keep prompts focused and single-purpose
   - Handle parameter defaults in function body
   - Use markdown for structured output

3. Error Handling:
   - Keep server startup code simple
   - Use try/except blocks appropriately
   - Print errors during development
   - Let FastMCP handle logging in production


