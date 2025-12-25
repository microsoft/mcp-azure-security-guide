# Development Best Practices for MCP Servers

## Overview

Building effective MCP servers requires different thinking than traditional API development. This page provides practical guidance for designing, implementing, and testing MCP tools that agents can successfully use.

---

## Tool Design Principles

The quality of your MCP tools directly impacts agent performance and user experience. Well-designed tools enable agents to accomplish tasks efficiently, while poorly designed tools lead to confusion, errors, and frustrated users.

This section covers the core principles for designing tools that agents can understand and use effectively.

!!! tip "Key Principle: Design for Agent Experience, Not API Completeness"

    Traditional APIs expose endpoints for flexibility. MCP tools should expose **tasks** that accomplish specific goals. Think "what does the user want to accomplish?" rather than "what operations does the system support?"

### Experience-Oriented vs System APIs

The most common mistake when building MCP servers is treating them like REST APIs: exposing every operation as a separate tool. This overwhelms LLMs and leads to poor agent performance.

❌ **Avoid**: Auto-generating tools from OpenAPI specs that expose every endpoint  
✅ **Prefer**: Task-oriented tools that encapsulate multi-step workflows

**Examples**:

| System API Approach | Experience-Oriented Approach |
|---------------------|------------------------------|
| `create_ticket`, `add_comment`, `assign_ticket`, `notify_user` | `file_support_request` (does all 4 steps internally) |
| `list_deployments`, `get_deployment`, `get_logs` | `summarize_recent_deployments` (aggregates and analyzes) |
| `search_orders`, `get_order_details`, `get_customer_info` | `lookup_order` (combines all 3, returns complete context) |

**Why this matters**: Each tool invocation consumes tokens and requires the LLM to reason about next steps. Combining related operations into single tools reduces complexity and improves reliability.

### Key Design Guidelines

These five guidelines translate the core principle into concrete practices. Apply them when designing any MCP tool to maximize agent success.

??? tip "1. Limit Tool Count"

    **Why**: LLMs perform significantly worse when choosing from large tool sets. Keep servers focused and specialized with a manageable number of tools.

    **If you have many operations**:
    
    - Split into multiple domain-specific servers (Sales, Finance, IT)
    - Group related operations into composite tools
    - Use tool parameters to handle variations (e.g., `status` parameter instead of `get_active_orders`, `get_pending_orders`)
    
    **Rule of thumb**: If you're exposing more than 10-15 tools from a single server, consider whether they can be consolidated or split across multiple servers.

??? tip "2. Task-Focused Naming"

    **Why**: Tool names should describe what the user accomplishes, not what the system does.

    **Examples**:

    - ❌ `execute_sql_query` → ✅ `analyze_sales_trends`
    - ❌ `get_tickets_by_status` → ✅ `find_open_support_issues`
    - ❌ `update_record` → ✅ `change_customer_address`

??? tip "3. Return Decision-Ready Data"

    **Why**: Agents should get actionable summaries, not raw data dumps they need to process.

    **Pattern**: Instead of returning 100 rows of deployment logs, return:
    
    ```json
    {
      "summary": "3 deployments in last 24h: 2 successful, 1 failed",
      "failed_deployment": {
        "name": "api-v2.1.3",
        "error": "Health check timeout",
        "recommendation": "Check database connection pool settings"
      },
      "recent_deployments": [...]
    }
    ```

??? tip "4. Embed Guardrails Server-Side"

    **Why**: Don't rely on LLMs to enforce business rules. Validate and constrain operations in your server implementation.

    **Examples**:
    
    - Validate refund amounts don't exceed order total
    - Prevent account deletion if balance > 0
    - Require manager approval for expenses > $5000
    - Enforce data residency rules based on tenant

??? tip "5. Write Clear Descriptions"

    **Why**: The description is the LLM's primary signal for when and how to use each tool.

    **Template**:
    
    ```json
    {
      "name": "file_support_request",
      "description": "Creates a new support ticket with customer details, issue description, and priority. Automatically assigns to appropriate team based on category. Use when customer reports a problem or needs help.",
      "inputSchema": {...}
    }
    ```

    **Best Practices**:
    
    - Start with the action verb ("Creates", "Retrieves", "Updates")
    - Explain what happens automatically
    - State when the tool should be used
    - Keep to 2-3 sentences maximum

### Common Anti-Patterns

Learn from these common mistakes:

??? warning "Exposing Raw Database Queries"

    **Anti-Pattern**: `execute_sql(query: string)` tool
    
    **Why it's bad**: 
    - Security risk (SQL injection, data exposure)
    - LLMs aren't reliable SQL generators
    - No way to enforce access control

    **Instead**: Create purpose-specific tools (`get_customer_orders`, `search_products`)

??? warning "Too Many Similar Tools"

    **Anti-Pattern**: `get_active_orders`, `get_pending_orders`, `get_completed_orders`, `get_cancelled_orders`
    
    **Why it's bad**: Increases tool count, confuses LLMs
    
    **Instead**: `get_orders(status: enum)` - Use parameters for variations

??? warning "Returning Unstructured Text"

    **Anti-Pattern**: Returning HTML, markdown, or verbose prose
    
    **Why it's bad**: Agents can't extract structured data reliably
    
    **Instead**: Return JSON with clear fields the agent can reference

??? warning "No Rate Limiting or Quotas"

    **Anti-Pattern**: Allowing unlimited tool invocations
    
    **Why it's bad**: Agents can trigger runaway loops or exhaust backend resources
    
    **Instead**: Implement rate limits per user/client in Azure APIM

---

## Schema Design

Schemas are how you communicate with the LLM about what your tool needs and what it returns. Well-designed schemas help the LLM understand exactly what parameters to provide and what to expect back. Poorly designed schemas lead to validation errors, confused agents, and failed invocations.

Think of schemas as the contract between your tool and the agent using it.

### Input Schema (Required)

Every tool **must** define an `inputSchema` using JSON Schema. This tells the LLM what parameters the tool expects and how to use them correctly.

**MCP Requirements**:

- `inputSchema` is **required** and must be a valid JSON Schema object (not `null`)
- Defaults to JSON Schema 2020-12 if no `$schema` field is present
- For tools with no parameters, use: `{"type": "object", "additionalProperties": false}`
- Tool names should be 1-128 characters, case-sensitive, using only: `A-Z`, `a-z`, `0-9`, `_`, `-`, `.`

**Key principle**: Keep schemas simple and self-documenting. Rich descriptions help the LLM understand not just *what* parameters exist, but *when* and *how* to use them.

??? example "Good Input Schema"

    ```json
    {
      "type": "object",
      "properties": {
        "customer_email": {
          "type": "string",
          "format": "email",
          "description": "Customer's email address for communication and ticket tracking"
        },
        "issue_type": {
          "type": "string",
          "enum": ["billing", "technical", "account"],
          "description": "Category of the support request. Use 'billing' for payment issues, 'technical' for product problems, 'account' for login or access issues."
        },
        "priority": {
          "type": "string",
          "enum": ["low", "normal", "high", "urgent"],
          "default": "normal",
          "description": "Urgency level. Use 'urgent' only for system outages affecting multiple users. Default is 'normal' for standard requests."
        },
        "description": {
          "type": "string",
          "description": "Detailed description of the issue in the customer's own words. Include relevant error messages or symptoms."
        }
      },
      "required": ["customer_email", "issue_type", "description"]
    }
    ```

    **What makes this effective**:
    
    - **Enums constrain choices**: Prevents invalid values and helps LLM select the right category
    - **Rich descriptions**: Explains when and how to use each option with specific examples
    - **Required fields**: Clearly marked so LLM knows what must be provided
    - **Sensible defaults**: Reduces decision burden for common cases
    - **Helpful context**: Descriptions guide the LLM toward correct usage

### Output Schema (Optional)

Tools may optionally define an `outputSchema` to validate structured results. Think of this as documentation for the LLM about what to expect back, plus a validation layer to catch issues early.

**When provided**:

- Servers **must** return structured results conforming to this schema
- Clients **should** validate results against the schema
- Results can include both `content` (unstructured) and `structuredContent` (structured) fields

**Why use output schemas?**: They enable strict validation, provide type information for better integration, and help LLMs understand how to parse and use your tool's responses.

### Output Content Guidelines

Structure matters. The way you format responses directly affects whether agents can extract and use the information correctly.

**The golden rule**: Return structured data that agents can reason about, not HTML or verbose text that requires parsing.

Most tools should return simple JSON with clear, extractable fields. For specialized use cases, MCP supports additional content types like base64-encoded images or resource links, but start with structured JSON unless you have a specific need for richer content.

❌ **Avoid**: Unstructured responses that bury information in prose
```json
{
  "message": "Ticket #12345 created successfully. Assigned to IT team. Customer will be notified via email."
}
```

✅ **Prefer**: Structured responses with clear, extractable fields
```json
{
  "ticket_id": "12345",
  "status": "open",
  "assigned_team": "IT",
  "estimated_response_time": "2 hours",
  "next_steps": [
    "Customer notified via email",
    "IT team alerted",
    "Agent will follow up if no response in 2 hours"
  ]
}
```

**Why this matters**: The second format lets the agent extract specific values (`ticket_id`, `status`) and reason about next steps. The first format requires parsing prose, which is error-prone and unreliable.

---

## Error Handling

Errors are inevitable, but how you handle them determines whether an agent can recover or gets stuck. The difference between a frustrating failure and a successful retry often comes down to how clearly you communicate what went wrong and what to do about it.

Think of error messages as teaching moments for the LLM. A good error message doesn't just say "something failed"—it explains the problem, provides context, and suggests a path forward.

### Two Types of Errors

MCP distinguishes between errors that the agent can potentially fix and errors that indicate deeper problems:

**Tool Execution Errors** (`isError: true`) are for problems the LLM can understand and correct:

```json
{
  "content": [
    {
      "type": "text",
      "text": "User lacks permission to approve expenses over $5000. Required role: Finance.Approver. Your roles: Employee. Suggested action: Request approval from finance team."
    }
  ],
  "isError": true
}
```

**Why this works**: The error explains what failed (permission check), what was expected (Finance.Approver role), what the user has (Employee role), and what to do next (request approval). The LLM can use this information to adjust its approach or inform the user.

**Protocol Errors** (JSON-RPC) are for structural problems like malformed requests or unknown tools. These indicate issues the LLM is unlikely to fix, such as calling a tool that doesn't exist or passing invalid JSON.

### Writing Effective Error Messages

The best error messages follow a simple pattern: **What happened** → **Why it happened** → **What to do next**

??? example "Good Error Messages"

    **Rate Limit**:
    ```
    "Rate limit exceeded: 100 requests per minute. Current usage: 103 requests. 
    Wait 45 seconds before retrying or reduce request frequency."
    ```

    **Validation Error**:
    ```
    "Invalid date format in 'start_date' field. Received: '2024/12/23'. 
    Expected format: YYYY-MM-DD (e.g., '2024-12-23')."
    ```

    **Permission Error**:
    ```
    "Insufficient permissions to delete customer records. Required role: Admin. 
    Your roles: Viewer, Editor. Contact your administrator to request Admin access."
    ```

    **Resource Not Found**:
    ```
    "Order #12345 not found. Verify the order ID is correct. 
    Recent orders: #12344, #12346, #12347."
    ```

    **What makes these effective**:
    - State the problem clearly
    - Provide specific context (what was received, what was expected)
    - Offer actionable next steps
    - Include helpful details (like similar valid values)

??? example "Poor Error Messages"

    ❌ `"Error"`  
    ❌ `"Invalid input"`  
    ❌ `"Permission denied"`  
    ❌ `"Not found"`

    **Why these fail**: No context about what went wrong, no guidance on how to fix it. The agent has no path forward.

### Security Considerations

!!! warning "Balance Helpfulness with Information Disclosure"

    Error messages should help agents recover **without exposing sensitive system details**. Always ask: "Could an attacker use this information?"

    **Never expose in error messages**:
    
    - Internal file paths, stack traces, or database schemas
    - Other users' data (emails, names, account details)
    - Whether specific resources exist (prevents enumeration attacks)
    - API keys, connection strings, or credentials
    - Detailed system architecture or service names

    **Safe to include**:
    
    - What operation failed ("refund request", "ticket creation")
    - Why it failed at a business logic level ("insufficient permissions", "invalid date format")
    - What the agent should do next ("wait 45 seconds", "request approval")
    - Request/correlation IDs for support teams (sanitized)

    **Pattern**: Log detailed errors server-side for debugging, but return sanitized, actionable messages to agents.

??? example "Secure vs Insecure Error Messages"

    ❌ **Information Disclosure**:
    ```
    "User john.doe@company.com does not exist in database table 'users' 
    (connection: sql-prod-eastus.database.windows.net)"
    ```
    **Problems**: Reveals database schema, connection string, confirms which users don't exist (enumeration)

    ✅ **Secure Alternative**:
    ```
    "Unable to process request. Verify the email address and try again. 
    Request ID: req-abc-123"
    ```
    **Why it works**: Doesn't confirm whether user exists, provides support reference, no internal details

    ---

    ❌ **Excessive Technical Detail**:
    ```
    "NullReferenceException at OrderService.cs:142 in GetCustomerOrders(). 
    Stack trace: [...]"
    ```
    **Problems**: Exposes code structure, file paths, implementation details

    ✅ **Secure Alternative**:
    ```
    "Unable to retrieve orders at this time. Please try again or contact support. 
    Request ID: req-xyz-789"
    ```
    **Why it works**: Acknowledges failure, provides path forward, includes support reference

### Include Tracing Information

For operations teams debugging issues, include **sanitized** request IDs or correlation tokens:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Unable to create ticket due to temporary service issue. Request ID: req-abc-123. Please retry in a few moments."
    }
  ],
  "isError": true
}
```

Log detailed context **server-side only** (stack traces, input parameters, user identity, internal service names). Keep error messages to agents focused on what they need to recover, not what you need to debug.

---

## Testing MCP Servers

Testing MCP servers is fundamentally different from testing traditional APIs. With REST APIs, you verify that endpoints return correct responses for given inputs. With MCP servers, you also need to verify that LLMs can **understand** and **correctly use** your tools. This means testing not just functionality, but discoverability, clarity, and agent behavior.

A tool that works perfectly in isolation can still fail if its description is ambiguous, its schema is unclear, or the LLM chooses it at the wrong time. Effective MCP testing validates the entire interaction chain: from tool discovery, through parameter selection, to result interpretation.

### Three Layers of Testing

Think of MCP testing as a pyramid with three layers, each testing different aspects of your implementation:

```
                    ┌─────────────────────────────────┐
                    │    🎯 AGENT TESTS               │
                    │  Validate LLM Behavior          │
                    │  • Real-world scenarios         │
                    │  • Tool selection & workflows   │
                    │  • Slowest, most valuable       │
                    └─────────────────────────────────┘
                             ▲
                             │
          ┌──────────────────────────────────────────────┐
          │        ⚙️  PROTOCOL TESTS                    │
          │     Verify MCP Compliance                    │
          │     • Tool discovery & invocation            │
          │     • Response format & error handling       │
          │     • Moderate speed                         │
          └──────────────────────────────────────────────┘
                             ▲
                             │
     ┌────────────────────────────────────────────────────────┐
     │              🔧 UNIT TESTS                             │
     │           Test Tool Logic                              │
     │           • Business rules & validation                │
     │           • Input/output correctness                   │
     │           • Fast, deterministic                        │
     └────────────────────────────────────────────────────────┘
```

**Layer 1: Unit Tests** verify your tool logic works correctly  
**Layer 2: Protocol Tests** verify you implement MCP correctly  
**Layer 3: Agent Tests** verify LLMs can successfully use your tools

Each layer builds on the previous one. Skip unit tests and you'll waste time debugging agent failures caused by basic logic errors. Skip agent tests and you'll ship tools that work in theory but fail in practice.

### Unit Tests: Tool Logic

At this layer, test your tool implementation as pure functions. Mock external dependencies (databases, APIs) and verify that given specific inputs, you get expected outputs.

**What to test**:

- **Input validation**: Required fields are enforced, invalid values rejected, edge cases handled
- **Business logic**: Rules execute correctly (refund limits, permission checks, data transformations)
- **Output structure**: Results match your schema, include all required fields
- **Error conditions**: Graceful handling of failures (network errors, missing resources, rate limits)

**Why this matters**: Unit tests are fast and deterministic. They catch logic bugs before you even start the server, saving time in later testing stages.

??? example "Unit Test Examples"

    ```python
    def test_file_support_request_creates_ticket():
        """Test basic ticket creation flow"""
        result = file_support_request(
            customer_email="test@example.com",
            issue_type="billing",
            description="Charged twice"
        )
        assert result["ticket_id"] is not None
        assert result["assigned_team"] == "Finance"
        assert result["status"] == "open"
    
    def test_file_support_request_validates_email():
        """Test input validation"""
        with pytest.raises(ValueError, match="Invalid email format"):
            file_support_request(
                customer_email="not-an-email",
                issue_type="billing",
                description="Issue"
            )
    
    def test_file_support_request_handles_api_timeout():
        """Test error handling"""
        with mock.patch('ticketing_api.create', side_effect=TimeoutError):
            result = file_support_request(
                customer_email="test@example.com",
                issue_type="technical",
                description="Can't login"
            )
            assert result["isError"] is True
            assert "retry" in result["content"][0]["text"].lower()
    ```

### Protocol Tests: MCP Compliance

At this layer, test that your server correctly implements the MCP specification. This includes proper JSON-RPC message handling, correct tool schema format, and conformance to protocol requirements.

**What to test**:

- **Tool discovery**: `tools/list` returns correctly formatted tool definitions with valid schemas
- **Tool invocation**: `tools/call` accepts requests, executes tools, returns proper response format
- **Error responses**: Protocol errors use correct JSON-RPC error codes and structure
- **Authentication**: JWT validation, unauthorized requests rejected, user context propagated

**Why this matters**: Protocol compliance ensures your server works with any MCP client. These tests catch structural issues like malformed JSON schemas or incorrect response formats that would cause client-side failures.

??? example "Protocol Test Examples"

    ```python
    def test_tools_list_returns_valid_schemas():
        """Verify tool definitions conform to MCP spec"""
        response = client.request("tools/list")
        assert "tools" in response
        
        for tool in response["tools"]:
            assert "name" in tool
            assert "description" in tool
            assert "inputSchema" in tool
            
            # Verify schema is valid JSON Schema
            validate_json_schema(tool["inputSchema"])
    
    def test_tools_call_returns_correct_format():
        """Verify tool invocation returns proper structure"""
        response = client.call_tool(
            "file_support_request",
            {
                "customer_email": "test@example.com",
                "issue_type": "billing",
                "description": "Issue"
            }
        )
        
        assert "content" in response
        assert isinstance(response["content"], list)
        assert all("type" in item for item in response["content"])
    
    def test_unauthorized_request_rejected():
        """Verify authentication is enforced"""
        client_no_auth = MCPClient(server_url, auth_token=None)
        
        with pytest.raises(ProtocolError, match="401"):
            client_no_auth.call_tool("file_support_request", {...})
    ```

    **Tools for protocol testing**:
    
    - **MCP Inspector**: Visual tool for testing MCP servers interactively
    - **MCP SDK test utilities**: Built-in helpers for protocol conformance testing
    - **JSON Schema validators**: Verify your tool schemas are valid

### Agent Tests: Real-World Usage

At this layer, test with actual LLM agents to validate that your tools work in practice. This is where you discover if tool descriptions are clear, if the LLM chooses the right tools, and if multi-step workflows succeed.

**What to test**:

- **Tool selection**: Given a user request, does the agent choose the right tool?
- **Parameter extraction**: Does the agent correctly extract parameters from user input?
- **Multi-step workflows**: Can the agent chain multiple tool calls to complete complex tasks?
- **Error recovery**: When a tool fails, does the agent retry appropriately or inform the user?

**Why this matters**: This is the only way to validate the **user experience**. A tool can be technically correct but unusable if the LLM misunderstands when to call it or how to interpret results.

??? example "Agent Test Examples"

    ```python
    def test_agent_files_billing_complaint():
        """Test end-to-end ticket creation from user request"""
        agent = create_test_agent(tools=["file_support_request"])
        
        response = agent.process(
            "Customer jane@example.com was charged twice for order #5678"
        )
        
        # Verify agent called the right tool
        assert "file_support_request" in response.tool_calls
        
        # Verify parameters extracted correctly
        call = response.tool_calls["file_support_request"]
        assert call["customer_email"] == "jane@example.com"
        assert call["issue_type"] == "billing"
        assert "charged twice" in call["description"]
    
    def test_agent_handles_ambiguous_request():
        """Test how agent handles unclear user input"""
        agent = create_test_agent(tools=["file_support_request"])
        
        response = agent.process("I need help")
        
        # Agent should ask clarifying questions, not make assumptions
        assert any(
            keyword in response.text.lower() 
            for keyword in ["what", "which", "tell me more"]
        )
    
    def test_agent_recovers_from_rate_limit():
        """Test error recovery behavior"""
        agent = create_test_agent(tools=["lookup_order"])
        
        # First call succeeds, second hits rate limit, third succeeds
        with mock_rate_limiting(limit=1):
            response = agent.process(
                "Look up order #1234, then order #5678"
            )
        
        # Verify agent retried after delay
        assert response.success
        assert len(response.tool_calls) == 3  # Initial + retry + second order
    ```

    **Example test scenarios**:
    
    - **Happy path**: "File a billing issue for customer@example.com who was charged twice"
    - **Ambiguous input**: "I have a problem" (does agent ask clarifying questions?)
    - **Multi-step task**: "Find all open tickets assigned to IT and summarize common issues"
    - **Error handling**: "Create a ticket for invalid-email" (does agent handle validation errors?)
    - **Edge cases**: "File an urgent ticket for a system-wide outage affecting all users"

### Testing Strategy

!!! tip "Build Quality Layer by Layer"

    **🔧 Start with unit tests** to ensure tool logic is solid
    
    - Fast and catch basic bugs early
    - Test business rules, validation, error handling
    - Run on every commit
    
    **⚙️ Add protocol tests** once tools work individually
    
    - Verify MCP compliance and catch integration issues
    - Test tool discovery, invocation, response format
    - Run on every commit
    
    **🎯 Finish with agent tests** to validate real-world usage
    
    - Slower and more complex, but catch usability issues
    - Test tool selection, parameter extraction, workflows
    - Run on pull requests or nightly builds

**Debugging workflow**: If agent tests fail, first check if unit and protocol tests pass. If they do, the problem is likely with tool descriptions, schema clarity, or the number of tools (LLM overwhelmed by choices). If lower-layer tests fail, fix those first before returning to agent tests.

---

## Reference Documentation

### MCP Specification

- [MCP Schema Design](https://modelcontextprotocol.io/specification/2025-06-18/schema) - Official JSON Schema usage guidelines for tools
- [Tools Specification](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) - Complete reference for tool definitions, schemas, and error handling
- [Server Implementation](https://modelcontextprotocol.io/specification/2025-11-25/server) - Core server features and capabilities
- [Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) - How to return resource links and embedded resources from tools

### Tool Design Best Practices

- [Anthropic: Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) - Comprehensive guide on designing effective tools for LLM agents

---

## Related Security Topics

- [MCP03 - Tool Poisoning](../mcp/mcp03-tool-poisoning.md) - Malicious tool definitions and descriptions
- [MCP05 - Command Injection](../mcp/mcp05-command-injection.md) - Unsafe tool parameter handling
- [MCP06 - Prompt Injection](../mcp/mcp06-prompt-injection.md) - Manipulating agent behavior via tool responses
- [MCP08 - Lack of Audit & Telemetry](../mcp/mcp08-telemetry.md) - Monitoring and logging best practices
- [MCP10 - Context Oversharing](../mcp/mcp10-context-oversharing.md) - Information disclosure in tool outputs

---

## Next Steps

- **Deciding what to build?** → [When to Use MCP](when-to-use-mcp.md) for decision framework
- **Wrapping existing APIs?** → [Migration Guidance](migration-guidance.md) for adapter patterns
- **Ready to deploy?** → [Deployment Patterns](deployment-architecture.md) for infrastructure strategies
- **Need governance controls?** → [Enterprise Patterns](enterprise-patterns.md) for organizational guidance
- **Securing your implementation?** → [OWASP MCP Top 10](../index.md#owasp-mcp-top-10) for security best practices
