# FastMCP Python SDK Guide

FastMCP is a framework for creating Model Context Protocol (MCP) servers in Python. This guide covers the key concepts and patterns for developing with FastMCP based on the official examples.

## Basic Concepts

### Server Creation

```python
from mcp.server.fastmcp import FastMCP

# Create an MCP server with a name
mcp = FastMCP("Demo")
```

### Tools and Resources

FastMCP supports two main types of endpoints:
1. Tools - Functions that perform actions
2. Resources - URI-based data access points

## Defining Tools

### Basic Tool Definition

```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

### Tools with Parameter Descriptions

Use Pydantic's Field for adding parameter descriptions and validation:

```python
from pydantic import Field

@mcp.tool()
def greet_user(
    name: str = Field(description="The name of the person to greet"),
    title: str = Field(description="Optional title like Mr/Ms/Dr", default=""),
    times: int = Field(description="Number of times to repeat greeting", default=1),
) -> str:
    """Greet a user with optional title and repetition"""
    greeting = f"Hello {title + ' ' if title else ''}{name}!"
    return "\n".join([greeting] * times)
```

## Complex Input Types

### Using Pydantic Models

FastMCP integrates seamlessly with Pydantic for complex data validation:

```python
from typing import Annotated
from pydantic import BaseModel, Field

class ShrimpTank(BaseModel):
    class Shrimp(BaseModel):
        name: Annotated[str, Field(max_length=10)]
    
    shrimp: list[Shrimp]

@mcp.tool()
def name_shrimp(
    tank: ShrimpTank,
    extra_names: Annotated[list[str], Field(max_length=10)],
) -> list[str]:
    """List all shrimp names in the tank"""
    return [shrimp.name for shrimp in tank.shrimp] + extra_names
```

## Defining Resources

### Dynamic Resources

Resources use URI templates for dynamic access:

```python
@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """Get a personalized greeting"""
    return f"Hello, {name}!"
```

## Advanced Features

### Async Support

FastMCP supports async functions for both tools and resources:

```python
@mcp.tool()
async def async_operation():
    result = await some_async_function()
    return result
```

### Dependency Management

Specify external dependencies for your MCP server:

```python
mcp = FastMCP(
    "server_name",
    dependencies=[
        "pydantic-ai-slim[openai]",
        "asyncpg",
        "numpy",
    ],
)
```

### State Management

FastMCP can maintain state across requests using file system or databases:

```python
from pathlib import Path

PROFILE_DIR = (
    Path.home() / ".fastmcp" / os.environ.get("USER", "anon") / "state"
).resolve()
PROFILE_DIR.mkdir(parents=True, exist_ok=True)
```

### Error Handling

Use Python's standard exception handling or Pydantic validation:

```python
@mcp.tool()
def safe_operation(input: str) -> str:
    try:
        result = process_input(input)
        return result
    except ValueError as e:
        return f"Error processing input: {str(e)}"
```

## Best Practices

1. **Documentation**
   - Always include docstrings for tools and resources
   - Use Field descriptions for parameters
   - Document expected input/output formats

2. **Type Hints**
   - Always use type hints for parameters and return values
   - Use Pydantic models for complex types
   - Leverage Annotated types for additional validation

3. **Validation**
   - Use Pydantic Field for parameter validation
   - Implement proper error handling
   - Validate inputs before processing

4. **Resource Naming**
   - Use clear, descriptive URIs for resources
   - Follow REST-like patterns when appropriate
   - Document URI parameters

5. **State Management**
   - Use appropriate storage for state (files, databases)
   - Clean up resources properly
   - Handle concurrent access appropriately

## Common Patterns

### Parameter Validation
```python
from typing import Annotated
from pydantic import Field

@mcp.tool()
def validated_function(
    required_param: str,
    optional_param: str = Field(default=""),
    bounded_param: Annotated[int, Field(ge=0, le=100)] = 50,
):
    """Function with validated parameters"""
    pass
```

### Resource Patterns
```python
# Single item resource
@mcp.resource("item://{id}")
def get_item(id: str) -> dict:
    pass

# Collection resource
@mcp.resource("items://")
def get_items() -> list:
    pass

# Nested resource
@mcp.resource("user://{user_id}/items/{item_id}")
def get_user_item(user_id: str, item_id: str) -> dict:
    pass
```

### Async Database Operations
```python
@mcp.tool()
async def db_operation():
    async with pool.acquire() as conn:
        result = await conn.fetchrow("SELECT * FROM table")
        return result
```

## Error Handling Patterns

```python
from pydantic import ValidationError

@mcp.tool()
def safe_tool(input_data: dict) -> dict:
    try:
        # Validate input
        validated = InputModel(**input_data)
        
        # Process data
        result = process_data(validated)
        
        return {"status": "success", "data": result}
    except ValidationError as e:
        return {"status": "error", "message": "Invalid input", "details": str(e)}
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

This guide covers the core concepts and patterns found in the FastMCP Python SDK examples. Refer to the official documentation for more detailed information and advanced usage scenarios.