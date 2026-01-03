# Migration Guidance for Existing APIs

## Overview

Most organizations already have APIs, microservices, and integration endpoints. The question isn't whether to replace them with MCP—it's **when and how to expose existing capabilities through MCP** while maintaining existing integrations.

This page provides practical patterns for migrating or wrapping existing APIs to support MCP-based AI agent workflows.

!!! tip "Key Principle: MCP as an Adapter, Not a Replacement"

    Think of MCP as a translation layer that makes existing APIs consumable by AI agents. Your REST APIs, GraphQL endpoints, and microservices continue to operate as they do today. MCP servers sit in front of them, translating agent requests into API calls and responses back into agent-friendly formats.

---

## Migration Scenarios

Different starting points require different approaches. The table below maps your current state to a recommended migration pattern.

| **Current State** | **Recommended Approach** | **Effort** | **Key Considerations** |
|-------------------|-------------------------|-----------|------------------------|
| **Existing OpenAPI spec** | Build MCP adapter using spec as source of truth | **Low** | Use tools to auto-generate MCP tool descriptions from OpenAPI schema |
| **Selective endpoints** | Expose only AI-safe operations via filtered manifest | **Medium** | Decide which operations are read-only, which require approval, which are excluded |
| **New API for agents** | Design MCP-first with clear tool semantics | **Medium** | Opportunity to optimize for agent usability from the start |
| **Legacy APIs with inconsistent schemas** | Hold off until modernized or standardized | **High** | Fix the API design first; MCP won't solve poor API quality |

---

## Pattern 1: OpenAPI Wrapper

**When to use**: You have a well-documented REST API with an OpenAPI specification and want to expose some or all of its endpoints to AI agents.

??? info "How It Works"

    1. **Parse the OpenAPI spec** to extract endpoints, parameters, and response schemas
    2. **Generate MCP tool descriptions** that map each endpoint to a tool with natural-language semantics
    3. **Deploy an MCP server** that translates tool invocations into HTTP API calls
    4. **Apply filtering and governance** to control which endpoints are exposed

    ### Architecture

    ```
    ┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
    │  AI Agent    │──────▶│  MCP Server      │──────▶│  REST API        │
    │              │       │  (Adapter)       │       │  (Existing)      │
    └──────────────┘       └──────────────────┘       └──────────────────┘
                                  │
                                  │ Reads
                                  ▼
                           ┌──────────────────┐
                           │  OpenAPI Spec    │
                           │  (Single Source  │
                           │   of Truth)      │
                           └──────────────────┘
    ```

??? example "Azure Implementation"

    **Azure API Management as MCP Gateway**
    
    Use APIM policies to:

    - Expose a subset of APIs as MCP tools
    - Add rate limiting, caching, and transformation
    - Enforce authentication and authorization at the gateway layer
    - Route to backend APIs with header injection

??? tip "Benefits & Considerations"

    **Benefits**:
    
    - **Low effort**: Automated generation from existing documentation
    - **Single source of truth**: OpenAPI spec remains authoritative
    - **Gradual rollout**: Start with a few endpoints, expand over time

    **Considerations**:
    
    - Not all REST operations map cleanly to agent tools (e.g., streaming, file uploads)
    - Need to add natural-language descriptions if OpenAPI spec is sparse
    - Agent may not understand complex parameter dependencies

---

## Pattern 2: Selective Exposure

**When to use**: Your API has dozens of endpoints, but only a few are appropriate or safe for AI agents. You want to expose a curated subset with additional governance.

??? info "How It Works"

    1. **Identify AI-safe operations**: Read-only, non-destructive, well-documented
    2. **Create an MCP server** that exposes only these operations
    3. **Apply security controls**: Separate authentication, stricter rate limits, approval workflows for sensitive operations
    4. **Monitor and expand**: Add more operations as confidence grows

    ### Architecture

    ```
    ┌──────────────────┐
    │   Full REST API  │
    │   50 endpoints   │
    └────────┬─────────┘
             │
             │ Filters & Governance
             │
             ▼
    ┌──────────────────┐       ┌──────────────────┐
    │  MCP Server      │◀──────│  AI Agent        │
    │  (Filtered)      │       │                  │
    │  - 5 read tools  │       └──────────────────┘
    │  - 2 write tools │
    │    (w/ approval) │
    └──────────────────┘
    ```

??? example "Azure Implementation"

    **Governance-First Architecture**
    
    This pattern emphasizes **policy-driven filtering** rather than automatic generation:
    
    - Maintain an authoritative registry of approved tools with governance metadata
    - MCP server dynamically loads and exposes only tools that pass governance checks
    - Each tool declaration includes metadata: operation type (read/write), sensitivity level, and approval requirements
    
    **Identity & Authorization Strategy**
    
    Implement defense-in-depth with layered access controls:
    
    - **Entra ID App Roles & Scopes**: Define granular permissions (`Orders.Read`, `Orders.Write`) for tool-level authorization
    - **Token Validation**: MCP server validates both audience (was token issued for this server?) and scopes/roles (does token have required permissions for this operation?)
    - **Human Approval Workflows**: Route sensitive operations through approval gates before execution
    
    **Key Differentiator**: Unlike Pattern 1's automated approach, this pattern gives security teams explicit control over each exposed operation with runtime policy enforcement and approval gates.

??? tip "Benefits & Considerations"

    **Benefits**:
    
    - **Granular control**: Choose exactly what agents can access
    - **Risk mitigation**: Start with safe operations, expand carefully
    - **Human-in-the-loop**: Approval workflows for sensitive actions

    **Considerations**:
    
    - Requires ongoing maintenance as API evolves
    - Agents may request unavailable operations and get frustrated
    - Need clear documentation explaining what's available and why

---

## Pattern 3: MCP-First Design

**When to use**: You're building a new API specifically for AI agents or re-architecting an existing API.

??? info "How It Works"

    1. **Start with agent use cases**: What tasks should the agent accomplish?
    2. **Design tools with clear semantics**: Each tool has a single, well-defined purpose
    3. **Build the MCP server first**: Implement the tool interface
    4. **Deploy through Azure API Management**: Use APIM as a gateway for both MCP and REST clients
    5. **Optionally expose REST API**: If needed for traditional clients, add REST endpoints alongside MCP

    ### Architecture

    ```
    ┌──────────────────────────────────────┐
    │   MCP-First API Design               │
    │                                      │
    │  ┌────────────────────────────────┐  │
    │  │  MCP Server (Primary Interface)│  │
    │  │  - get_order                   │  │
    │  │  - create_order                │  │
    │  │  - cancel_order                │  │
    │  └────────────┬───────────────────┘  │
    │               │                      │
    │               ▼                      │
    │  ┌────────────────────────────────┐  │
    │  │  Business Logic Layer          │  │
    │  └────────────┬───────────────────┘  │
    │               │                      │
    │               ▼                      │
    │  ┌────────────────────────────────┐  │
    │  │  Data Layer (Cosmos DB, SQL)   │  │
    │  └────────────────────────────────┘  │
    └──────────────────────────────────────┘
    ```

??? example "Azure Implementation"

    **Design Principles for Agent-Optimized Tools**
    
    - **Clear naming**: Use simple, action-oriented tool names that describe what the tool does
    - **Natural language descriptions**: Write descriptions that help LLMs understand when and how to use each tool
    - **Consistent schemas**: Define predictable input/output formats using JSON Schema
    - **Single responsibility**: Each tool should accomplish one well-defined task
    
    **Azure API Management as Gateway**
    
    Always deploy MCP servers behind Azure API Management, regardless of whether you need REST support:
    
    - **MCP-only scenarios**: APIM acts as a passthrough gateway, providing security, rate limiting, monitoring, and token validation without protocol translation
    - **Hybrid scenarios**: APIM exposes both MCP and REST interfaces to the same backend, applying consistent policies across protocols
    
    **Security & Identity Architecture**
    
    - **Microsoft Entra ID**: Authenticate both the MCP server and connecting clients
    - **APIM policy enforcement**: Validate tokens, enforce rate limits, and apply conditional access at the gateway
    - **Managed identities**: Use for backend data access (databases, storage, other Azure services)
    - **Application Insights**: Centralize logging and telemetry for both APIM and backend services

??? tip "Benefits & Considerations"
    - **Optimized for agents**: No retrofitting or translation overhead
    - **Simpler architecture**: Single interface to maintain
    - **Better agent experience**: Tools designed for natural-language invocation

    ### Considerations
    - Requires team buy-in to MCP-first approach
    - May need to support REST clients eventually
    - Fewer established patterns and examples (emerging space)

---

## Pattern 4: Legacy API Modernization

**When to use**: Your API is a legacy system with inconsistent schemas, poor documentation, or complex interdependencies.

### Recommendation

??? danger "Do NOT attempt to MCP-enable directly"

    **Instead**: 

    1. **Modernize the API first**:
        - Standardize schemas (use JSON Schema or OpenAPI)
        - Add comprehensive documentation
        - Simplify complex operations
        - Improve error handling

    2. **Then apply Pattern 1 or 2**: Once the API is clean and well-documented, wrap it with an MCP adapter

    3. **Or build a facade**: Create a new MCP-first API (Pattern 3) that internally calls the legacy system

    **Why This Approach?**

    - **MCP won't fix API quality issues**: An LLM can't understand a poorly designed API better than a human can
    - **Security risks**: Inconsistent APIs are harder to govern and easier to exploit
    - **Wasted effort**: You'll spend more time debugging agent behavior than improving the underlying system

---

## Azure-Specific Considerations

When migrating existing APIs to MCP on Azure, these four areas require special attention. Each presents common challenges and Azure-native solutions.

??? tip "1. Authentication & Identity"

    **Challenge**: Existing APIs may use API keys, JWT tokens, or custom auth schemes. 

    **Solution**:

    - Use **Azure API Management** to translate between authentication schemes
    - Backend API uses service principals or managed identities
    - MCP clients authenticate with Entra ID, APIM handles token exchange

??? tip "2. Rate Limiting & Quotas"

    **Challenge**: AI agents can generate high request volumes, potentially overwhelming backend APIs.

    **Solution**:

    - Apply rate limiting policies in APIM based on client identity
    - Use caching features in APIM (traditional and semantic) to cache frequent queries
    - Monitor with Application Insights and set alerts for anomalies

??? tip "3. Data Transformation"

    **Challenge**: Backend APIs may return verbose, nested, or legacy formats that aren't agent-friendly.

    **Solution**:

    - Use APIM transformation policies to simplify responses
    - MCP server can perform schema mapping (e.g., flatten nested objects)
    - Return only fields relevant to the agent use case

??? tip "4. Versioning & Rollout"

    **Challenge**: Backend APIs evolve over time. How to manage MCP tool versions?

    **Solution**:

    - Use API versioning in APIM (`/v1/`, `/v2/`)
    - MCP server can expose multiple tool versions (`get_order_v1`, `get_order_v2`)
    - Gradually deprecate old versions with client notifications

---

## Migration Checklist

  :material-check: **Assess current API quality**: Is it well-documented? Consistent? Secure?  
  :material-check: **Identify AI-safe operations**: Which endpoints are read-only or low-risk?  
  :material-check: **Choose migration pattern**: OpenAPI wrapper, selective exposure, or MCP-first?  
  :material-check: **Implement authentication**: Integrate with Microsoft Entra ID  
  :material-check: **Add governance controls**: Rate limiting, monitoring, approval workflows  
  :material-check: **Test with sample agents**: Validate tool descriptions and behavior  
  :material-check: **Deploy incrementally**: Start with 1-2 tools, expand based on feedback  
  :material-check: **Monitor and iterate**: Use Application Insights to track usage and errors  
  :material-check: **Document for developers**: Explain what's available and how to use it  

---

## Next Steps

- **Need help deciding?** → [When to Use MCP](when-to-use-mcp.md) for decision framework
- **Want to learn from real-world examples?** → [Enterprise Patterns & Lessons Learned](enterprise-patterns.md)
- **Ready to secure your implementation?** → [OWASP MCP Top 10](../index.md#owasp-mcp-top-10)
