# MCP10: Context Injection & Over-Sharing

### Azure Implementation: PARTIAL

![MCP10 Scenario](../images/mcp10-scenario.png)

> **Real-World Scenario**: The Cross-Tenant Leak
>
> A SaaS company operates a multi-tenant MCP server where multiple customers share the same infrastructure. A sales representative from Company A requests a summary of their sales pipeline. Due to a flaw in session handling, Company A’s context—including customer names, deal sizes, and pricing—is mistakenly associated with Company B’s session ID.
>
> Later, when an employee from Company B submits an unrelated request, the MCP server retrieves and returns Company A’s confidential sales data in the response. A single session management error results in a cross-tenant data breach, exposing sensitive information across organizational boundaries.
>
> **Think of it like**: A hotel where electronic room keys occasionally get mixed up. You swipe your card and, instead of entering your room, you walk into a stranger’s room with all their belongings visible. The system believes you belong there, so it grants full access.

## Understanding the Risk

MCP servers maintain *context* as working memory that includes conversation history, retrieved data, tool outputs, and intermediate results. When context isolation fails, information from one user, session, or tenant can be returned to another. In multi-tenant MCP systems, this failure can expose customer data, PII, intellectual property, or confidential business information and often without any malicious intent or external attack.

## The Azure Solution

Preventing cross-tenant context leakage requires strong isolation at every layer where context is stored or processed. Detection alone is insufficient.

> **Important**: Azure does not provide built-in semantic understanding of who data belongs to. Preventing cross-tenant leakage is primarily an architecture responsibility

**Response inspection as a safety net**  
Azure AI Content Safety PII detection can be used as a last-resort signal to identify and redact sensitive data before responses are returned. This helps limit impact but must not be relied on as the primary protection.

**Session and context isolation**  
Azure Cache for Redis should use strict key prefixes (for example, {tenantId}:{userId}:{sessionId}:\*) and short TTLs (such as 30 minutes) to prevent stale or shared context from persisting across sessions.

**Storage-level tenant separation**  
Azure Cosmos DB partitioning with hierarchical partition keys (for example, /tenantId/userId/sessionId) enforces isolation at the data layer, ensuring context from different tenants cannot be co-mingled or queried together.

**Gateway-level tenant identification**  
API Management subscription keys or tokens can be used to reliably associate incoming requests with a specific tenant, ensuring tenant identity is consistently propagated through the system.

**Network isolation for high-assurance environments**  
For workloads with strict isolation requirements:

- Deploy separate VNets per tenant
- Use Private Endpoints per tenant for services such as Cosmos DB and Redis
- Consider Azure Dedicated Host for regulated industries requiring physical isolation

**Key Takeaways**:

- Design for strict context isolation across tenants, users, and sessions
- Treat PII detection as a safety net, not a primary control
- Use tenant-scoped keys and TTLs for all session and context storage
- Enforce tenant isolation at the storage and network layers
- Assume application bugs will happen and design isolation accordingly

---

## Next Steps

- **Related risks**: [MCP01: Token Mismanagement](mcp01-token-mismanagement.md) | [MCP02: Privilege Escalation](mcp02-privilege-escalation.md)
- **Monitoring**: [MCP08: Lack of Audit & Telemetry](mcp08-telemetry.md) to detect context boundary violations
- **Strategic guidance**: [Enterprise Patterns & Lessons Learned](../adoption/enterprise-patterns.md) for data classification practices
- **Back to**: [OWASP MCP Top 10](../index.md#owasp-mcp-top-10)