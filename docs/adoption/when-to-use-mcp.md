# When to Use MCP

## Overview

The Model Context Protocol (MCP) is a powerful standard, but it's not the right choice for every API or integration. This page helps you decide when MCP adds value, and when traditional APIs or other approaches are more appropriate.

> **Think of it like this**: MCP is the "AI Integration Layer"
>
> Just as REST APIs standardized how applications communicate with each other, MCP standardizes how AI systems discover, understand, and interact with tools and data sources. But standardization has a cost: added complexity, new security considerations, and governance overhead.
>
> The key question: **Does the solution need to dynamically discover and reason about this capability, or is it better to hardcode the integration?**

## Decision Framework

Before MCP-enabling an API or tool, ask:

1. **Will an AI agent need to discover and invoke this without hardcoded integration?**
2. **Does the operation have clear semantic meaning that an LLM can understand?**
3. **Is this part of a broader agent workflow involving multiple tools?**
4. **Do you need interoperability across different AI frameworks and clients?**

If you answer "yes" to most of these, MCP is likely a good fit. If most answers are "no," consider starting with a traditional API.

---

## When to MCP-Enable APIs and Tools

??? success "1. Discoverable and Usable by AI Agents or Copilots"

    Use MCP when your API should be automatically discoverable, described, and callable by an AI model without requiring developers to write custom integration code for each AI system.

    **Example**: An API that exposes support tickets. With MCP, an AI agent can browse, query, and summarize tickets without hardcoded integrations. The agent discovers the tool at runtime, reads its description, and invokes it based on user intent.

    **Benefits**:
    
    - AI systems can reason about when and how to use your tool
    - No need to maintain custom connectors for Claude, ChatGPT, Copilot, etc.
    - Tools become part of the agent's dynamic capability set

    **Azure Implementation**: Expose Azure Functions, Logic Apps, or custom APIs as MCP servers. Use Azure API Management to provide a governed discovery layer.

??? success "2. Standardized Interoperability"

    Use MCP when you need to work with multiple AI clients and frameworks and want to avoid building custom connectors or integrations for each one.

    **Example**: Your organization uses GitHub Copilot, Microsoft Copilot Studio, and custom AI agents. Instead of building three separate integrations, you expose a single MCP server that all systems can consume.

    **Benefits**:
    
    - Write once, integrate everywhere
    - Future-proof against new AI clients entering your environment
    - Centralized governance and security posture

    **Azure Implementation**: Deploy MCP servers behind Azure API Management with consistent authentication (Microsoft Entra ID), rate limiting, and monitoring across all consumers.

??? success "3. Integration with Ecosystem That Supports Tool Integration"

    Use MCP when your capability is part of a broader agent workflow that involves other tools like search, calendar, reports, tickets, or orders.

    **Example**: An expense approval workflow where the agent needs to:
    
    1. Retrieve expense data (MCP server for Finance API)
    2. Check policy compliance (MCP server for Policy Engine)
    3. Send approval notifications (MCP server for Messaging API)

    Agents can reason across systems in a consistent way when all capabilities speak the same protocol.

    **Benefits**:
    
    - Enables complex, multi-step orchestration
    - Agents can compose capabilities without custom glue code
    - Easier to add new tools to the workflow

    **Azure Implementation**: Deploy a set of specialized MCP servers (read-only, write-only, domain-specific) and allow agents to discover and chain them through Azure API Management.

??? success "4. Publish to Marketplace or Into Ecosystem"

    Use MCP when you want to share your tools with other teams, developers, or AI runtimes in a consistent, future-proof format.

    **Example**: An internal "MCP Registry" where different business units publish their approved tools. AI agents can discover available capabilities from the registry and invoke them with proper governance.

    **Benefits**:
    
    - Democratizes AI capabilities across your organization
    - Encourages reuse and discoverability
    - Centralized governance and approval process

    **Azure Implementation**: Use Azure API Management and an internal MCP registry with role-based access control.

??? success "5. Multi-Step Orchestration"

    Use MCP when the capability supports **"agentic" flows** where an AI solution needs to make decisions, invoke multiple tools in sequence, and adapt based on intermediate results.

    **Example**: "Analyze customer sentiment and create a follow-up task if negative." The agent needs to:
    
    1. Call sentiment analysis (MCP server)
    2. Interpret the result
    3. Conditionally create a task (MCP server)

    Traditional APIs require the orchestration logic to be hardcoded. MCP allows the agent to handle the decision-making.

    **Benefits**:
    
    - Flexibility for dynamic, context-aware workflows
    - Reduces need for brittle, hardcoded orchestration
    - Agents can adapt to changing conditions

    **Azure Implementation**: Deploy lightweight MCP servers for discrete operations. Use Azure Monitor and Application Insights to observe how agents compose these operations in practice.

---

## When NOT to MCP-Enable APIs and Tools

??? warning "1. Not Semantically Useful to an Agent"

    Avoid MCP when your API lacks semantic clarity and may have inconsistent schemas, missing descriptions, and unclear input/output contracts.

    **Why?** MCP thrives when endpoints have clear, well-documented, natural-language-friendly descriptions. If your API is poorly documented or uses inconsistent naming, the LLM won't understand how to use it correctly, leading to errors and unpredictable behavior.

    **Instead**: Fix the API design first. Standardize schemas, add clear descriptions, and ensure consistency. Then consider MCP.

??? warning "2. Already Available in an Existing MCP Server"

    Avoid MCP when the capability already exists in an established, trusted MCP server.

    **Why?** Duplicating functionality creates governance headaches, version conflicts, and confusion for AI clients trying to choose the "right" tool.

    **Instead**: Extend or federate the existing MCP server. Contribute to the shared capability rather than forking.

    **Example**: If there's already an approved "Document Summarizer" MCP server, don't create a second one. Add your document sources to the existing server or propose enhancements.

??? warning "3. Missing Use Case"

    Avoid MCP when you don't have a clear, validated use case for AI agents to invoke the API.

    **Why?** MCP introduces complexity, security considerations, and governance overhead. Building an MCP server "just in case" wastes resources and increases attack surface.

    **Instead**: Start with OpenAPI or REST. Wait until there's demonstrated demand from AI agents or Copilots. Then wrap the API with MCP.

    **Example**: An internal HR API that's only called by a single legacy application. Unless there's a plan to integrate it with an agent, there's no reason to MCP-enable it.

??? warning "4. Fine-Grained Control or Tight Latency Requirements"

    Avoid MCP when the API is part of a critical path with strict latency, performance, or determinism requirements.

    **Why?** MCP introduces additional layers (protocol translation, LLM decision-making, token limits). For high-frequency trading, real-time telemetry, or mission-critical systems, direct API calls are more reliable.

    **Instead**: Keep these APIs as traditional REST/gRPC. If needed, wrap them with a thin MCP layer for monitoring or observability, but don't route critical transactions through it.

    **Example**: A stock trading API that must execute in milliseconds. The overhead of LLM reasoning and MCP protocol negotiation is unacceptable.

??? warning "5. Strictly for Backend or Internal Use"

    Avoid MCP when the API is purely for internal microservices, management operations, or infrastructure automation with no AI interaction.

    **Why?** MCP is designed for AI agents, not for service-to-service communication. Internal APIs that will never be invoked by an LLM don't benefit from MCP's capabilities.

    **Instead**: Use standard REST, gRPC, or message queues. Reserve MCP for user-facing or agent-facing capabilities.

    **Example**: An internal logging pipeline, a Kubernetes controller API, or a database backup service. These are operational, not conversational.

---

## Decision Tree

```
┌─────────────────────────────────────────┐
│ Will an AI agent invoke this API?       │
└─────────────┬───────────────────────────┘
              │
      ┌───────┴────────┐
      │ NO             │ YES
      │                │
      ▼                ▼
  Use REST       ┌─────────────────────────────┐
  or gRPC        │ Does it have clear semantic │
                 │ descriptions & consistency? │
                 └─────────────┬───────────────┘
                               │
                       ┌───────┴────────┐
                       │ NO             │ YES
                       │                │
                       ▼                ▼
                 Fix API first   ┌───────────────────────┐
                 then revisit    │ Part of multi-tool    │
                                 │ workflow or ecosystem?│
                                 └─────────┬─────────────┘
                                           │
                                   ┌───────┴────────┐
                                   │ NO             │ YES
                                   │                │
                                   ▼                ▼
                             Consider MCP    MCP is a
                             for future      strong fit
```

---

## Summary

| **Use MCP When** | **Avoid MCP When** |
|------------------|---------------------|
| AI agents need to discover and invoke dynamically | API is purely internal/operational |
| You need interoperability across AI frameworks | Lacks clear semantic descriptions |
| Part of multi-tool orchestration workflows | Already covered by existing MCP server |
| Publishing to internal catalog or marketplace | No validated AI use case yet |
| Supports agentic, multi-step decision flows | Critical path with tight latency needs |

---

## Next Steps

- **Ready to proceed?** → [Migration Guidance](migration-guidance.md) for practical implementation patterns
- **Want to learn from others?** → [Enterprise Patterns & Lessons Learned](enterprise-patterns.md) for real-world adoption strategies
- **Need to secure your MCP servers?** → [OWASP MCP Top 10](../index.md#owasp-mcp-top-10) for comprehensive security guidance
