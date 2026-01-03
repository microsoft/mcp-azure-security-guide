# Deployment Patterns for MCP Servers

## Overview

This page provides technical guidance for deploying MCP servers in enterprise Azure environments. We cover transport options, implementation patterns, and operational considerations to help you build secure, scalable, and governable MCP infrastructure.

!!! tip "Deployment Recommendation: Use Remote HTTP-Based MCP Servers"

    While local stdio MCP servers are convenient for prototyping, they create credential sprawl, bypass enterprise identity and policy controls, and provide zero visibility. Remote MCP servers integrate with Microsoft Entra ID, enforce centralized policies via Azure API Management, and provide comprehensive monitoring.
    
    **Reserve stdio servers only for prototyping or truly local operations** (filesystem access, IDE integrations). For production use cases involving network APIs or organizational data, deploy remote MCP servers.

For organizational patterns and governance approaches, see [Enterprise Patterns & Lessons Learned](./enterprise-patterns.md).

---

## Transport Comparison: Stdio vs HTTP

MCP supports two primary transports. Understanding their tradeoffs is essential for choosing the right deployment pattern.

| Aspect | Stdio (Local) | HTTP (Remote) |
|--------|---------------|---------------|
| **Deployment** | Runs as local process on workstation | Deployed as centralized service |
| **Authentication** | Static API keys/PATs in config files | Microsoft Entra ID + OAuth |
| **Identity** | No user attribution | Full user identity and device context |
| **Policy Enforcement** | None (each workstation independent) | Centralized via API Management |
| **Monitoring** | None | Comprehensive (Application Insights, Log Analytics) |
| **Credential Management** | Manual distribution and rotation | Managed Identity, automatic token refresh |
| **Governance** | Ungoverned (users install arbitrary servers) | Controlled (approved servers only) |
| **Best For** | Prototyping, local file operations | Production, network API access, enterprise data |

### Stdio Challenges in Enterprise

Local stdio servers create several enterprise challenges:

- **Credential sprawl**: API keys stored in plaintext on every workstation ([MCP01](../mcp/mcp01-token-mismanagement.md))
- **No attribution**: Cannot link actions to users or devices ([MCP07](../mcp/mcp07-authz.md))
- **Policy bypass**: Cannot enforce DLP, rate limiting, or access controls ([MCP07](../mcp/mcp07-authz.md))
- **Zero visibility**: No logs or monitoring ([MCP08](../mcp/mcp08-telemetry.md))
- **Supply chain risk**: Unvetted packages installed via `npx`, `uvx`, or Docker ([MCP04](../mcp/mcp04-supply-chain.md))

According to [Astrix Security's State of MCP report](https://astrix.security/learn/blog/state-of-mcp-server-security-2025/), 88% of MCP servers require credentials, and 53% rely on long-lived static secrets, which makes stdio inappropriate for most enterprise scenarios.

### When to Use Each Transport

**Use stdio for**:

:material-check: Prototyping and learning  
:material-check: Local filesystem operations  
:material-check: IDE integrations (no network calls)  
:material-check: Personal tools with no org data  

**Use HTTP for**:

:material-check: Production deployments  
:material-check: Network API access  
:material-check: Organizational data access  
:material-check: Multi-user scenarios  
:material-check: Compliance requirements  

---

## Remote MCP Architecture

Remote MCP servers run as Azure services behind a gateway that handles authentication, policy, and routing:

```
┌─────────────────────────────────────────────────┐
│          AI Agent (Claude Desktop, VS Code      │
│       GitHub Copilot, Copilot Studio)           │
└────────────────────┬────────────────────────────┘
                     │ HTTP + OAuth/OIDC
                     │ (No local credentials)
                     ▼
┌─────────────────────────────────────────────────┐
│         Centralized MCP Gateway                 │
│      (Azure API Management)                     │
│  • Microsoft Entra ID authentication            │
│  • Rate limiting and throttling                 │
│  • Policy enforcement (data loss prevention)    │
│  • Centralized monitoring and logging           │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    ┌────────┐  ┌────────┐   ┌────────┐
    │ Sales  │  │Finance │   │   HR   │
    │  MCP   │  │  MCP   │   │  MCP   │
    │ Server │  │ Server │   │ Server │
    └───┬────┘  └───┬────┘   └───┬────┘
        │           │            │
        │ Managed   │ Managed    │ Managed
        │ Identity  │ Identity   │ Identity
        ▼           ▼            ▼
     [Azure      [Azure       [Azure
      Services]   Services]    Services]
```

### Key Benefits

This architecture solves the fundamental problems of stdio deployments while enabling enterprise-grade security and operations:

| Benefit | How Remote Architecture Delivers |
|---------|----------------------------------|
| **Identity-based access** | Entra ID authentication + Conditional Access, no static credentials |
| **Centralized policy** | API Management enforces rate limits, DLP, time/location restrictions |
| **Complete visibility** | All requests logged to Application Insights and Log Analytics |
| **Governance** | Only approved servers accessible, version control enforced |
| **Operational control** | CI/CD pipelines, blue/green deployments, centralized updates |

---

## Azure Deployment Patterns

The following patterns show how to implement remote MCP servers on Azure, from simple single-server deployments to complex multi-tenant architectures. Choose the pattern that matches your scale and governance requirements.

??? example "Pattern 1: Single Remote MCP Server"

    Deploy a single remote MCP server with Entra ID authentication and basic monitoring.

    **Architecture**:

    ```
    ┌──────────┐       ┌──────────────┐        ┌──────────────────┐
    │   User   │──────▶│  Entra ID    │──────▶ │  API Management  │
    │  Agent   │       │   (OAuth)    │        │  • Auth policies │
    └──────────┘       └──────────────┘        │  • Rate limiting │
                                               └────────┬─────────┘
                                                        │
                                                        ▼
                                               ┌────────────────────┐
                                               │   Container App    │
                                               │   (MCP Server)     │
                                               └─────────┬──────────┘
                                                         │ Managed
                                                         │ Identity
                                                         ▼
                                               ┌────────────────────┐
                                               │  Azure Services    │
                                               │  (Cosmos, Storage) │
                                               └────────────────────┘
    ```

    **Components**:

    | Component | Azure Service | Purpose |
    |-----------|---------------|---------|
    | MCP Server | Container Apps | HTTP transport, web endpoint |
    | Authentication | Entra ID | OAuth 2.0, JWT validation |
    | Gateway | API Management | Routing, policies, rate limiting |
    | Identity | Managed Identity | Server-to-service auth (no secrets) |
    | Monitoring | Application Insights | Logs, metrics, alerts |

    **Benefits**: Simple setup, eliminates credential sprawl, basic observability.

??? example "Pattern 2: Gateway with Multiple Servers"

    Deploy a centralized gateway that routes to multiple backend MCP servers. This pattern is detailed in [Enterprise Patterns - Centralized MCP Gateway](./enterprise-patterns.md#emerging-adoption-patterns).

    **Architecture**:

    ```
                      ┌─── API Management ───┐
    User → Entra ID → │  • Auth & Policy     │ ──┬─→ Sales MCP
                      │  • Rate Limiting     │   ├─→ Finance MCP
                      │  • Monitoring        │   ├─→ HR MCP
                      └──────────────────────┘   └─→ IT MCP
    ```

    **Benefits**: Centralized control, consistent policies, simplified client config.

    See [detailed gateway pattern](./enterprise-patterns.md#emerging-adoption-patterns) in Enterprise Patterns.

??? example "Pattern 3: Multi-Tenant Deployment"

    Deploy MCP servers that serve multiple tenants with data isolation.

    **Architecture Options**:

    - **Separate Subscriptions**: Complete isolation per tenant, managed via Azure Lighthouse
        - Use when: Strict compliance, separate billing, complete isolation required
    
    - **Shared Infrastructure**: Single server farm with tenant-aware routing and row-level security
        - Use when: Cost efficiency matters, strong data isolation can be enforced

    **Implementation Considerations**:

    - Pass tenant ID in JWT claims
    - Validate tenant ID on every request
    - Use row-level security in databases
    - Implement tenant-scoped Managed Identities
    - Monitor for cross-tenant leakage

    **Benefits**: Scales to many tenants, supports SaaS models, enforces data residency.

---

## Operational Considerations

Remote MCP deployments introduce operational complexity that should be planned for:

| Consideration | Impact | Mitigation |
|---------------|--------|------------|
| **Infrastructure complexity** | Requires deployment pipelines, monitoring, HA | Use managed services (Container Apps, APIM), start simple |
| **Multi-tenant security** | Risk of confused deputy, cross-user data leakage | Implement on-behalf-of flows, row-level security, extensive testing |
| **Versioning** | Centralized upgrades affect all users | Blue/green deployments, feature flags, API versioning |
| **Discovery** | No standard mechanism for agents to find servers | Deploy internal registry (Azure API Center), document catalog |
| **Network latency** | Remote calls slower than local stdio | Design tools to minimize round-trips, use regional deployment, cache where appropriate |

---

## Summary

For enterprise MCP deployments:

✅ **Default to remote HTTP servers** - Integrate with Entra ID, enforce centralized policy, enable complete monitoring  
✅ **Deploy behind API Management** - Centralized gateway for authentication, rate limiting, and routing  
✅ **Design experience-oriented tools** - See [Development Best Practices](./development-best-practices.md) for tool design guidance  
✅ **Start with read-only** - Prove security controls before adding write capabilities  
✅ **Use stdio only for local operations** - Prototyping, filesystem access, no network APIs  

!!! tip "Getting Started with Remote MCP"

    **Phase 1: Pilot** - Deploy single remote server for read-only use case  
    **Phase 2: Governance** - Establish approval process and server catalog  
    **Phase 3: Gateway** - Build centralized APIM infrastructure with monitoring  
    **Phase 4: Scale** - Migrate existing servers and expand approved catalog  
    **Phase 5: Mature** - Enable write operations with appropriate controls

---

## Related Security Topics

- [MCP01 - Token Mismanagement](../mcp/mcp01-token-mismanagement.md) - Credential security
- [MCP07 - Insufficient Authentication & Authorization](../mcp/mcp07-authz.md) - Identity and access control
- [MCP08 - Lack of Audit & Telemetry](../mcp/mcp08-telemetry.md) - Monitoring and logging
- [MCP09 - Shadow MCP Servers](../mcp/mcp09-shadow-servers.md) - Preventing unauthorized deployments

---

## Next Steps

- **Understanding deployment trade-offs?** → Review transport comparison and architecture diagrams above
- **Need tool design guidance?** → [Development Best Practices](development-best-practices.md) for building effective tools
- **Ready for governance?** → [Enterprise Patterns](enterprise-patterns.md) for organizational controls
- **Wrapping existing APIs?** → [Migration Guidance](migration-guidance.md) for adapter patterns
- **Securing your deployment?** → [OWASP MCP Top 10](../index.md#owasp-mcp-top-10) for security guidance
