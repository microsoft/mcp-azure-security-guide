# MCP08: Lack of Audit and Telemetry

### Azure Implementation: FULL

![MCP08 Scenario](../images/mcp08-scenario.png)

> **Real-World Scenario**: The Invisible Breach
>
> An attacker compromises an MCP server and spends three weeks quietly exfiltrating customer data. When the breach is finally discovered through an external report, the security team scrambles to understand what happened. But there are no logs showing which tools were called, what data was accessed, or which users were affected. They can't determine the scope of the breach, can't notify the right customers, and can't prove to regulators that they've contained the incident. The lack of visibility turns a manageable breach into a catastrophic one.
>
> **Think of it like**: A bank vault with no security cameras and no visitor log. Even if you eventually discover money is missing, you have no way to know when it happened, who took it, or how much is gone.

## Understanding the Risk

Without comprehensive audit and telemetry, MCP systems operate blindly. Security teams cannot detect attacks in progress, investigate incidents after they occur, demonstrate compliance to auditors, or establish baselines for normal behavior. The MCP specification emphasizes logging tool invocations, resource access, and sampling requests because these signals are essential for understanding *who did what, when, and why* in an MCP environment.

## The Azure Solution

Effective MCP security requires centralized, correlated, and MCP-aware telemetry across identity, application, and network layers.

**Centralized logging and correlation**  
Azure Log Analytics provide a single platform for ingesting logs from MCP servers, API Management, Entra ID, and underlying Azure services. Using Kusto Query Language (KQL), security teams can correlate identity events, tool invocations, and data access across the full request lifecycle.

**MCP-aware application telemetry**  
Application Insights with OpenTelemetry enables distributed tracing, capturing the complete path of a request from user input through tool execution and response. MCP servers should emit structured telemetry that includes MCP-specific attributes such as user_id, session_id, tool_name, and request parameters to support investigation and forensic analysis.

**Visibility and investigation dashboards**  
Azure Monitor Workbooks provide dashboards that surface tool usage patterns, authentication failures, error rates, and anomalous behavior. These views help security teams quickly distinguish normal activity from suspicious behavior.

**Detection and alerting**  
Alert rules trigger notifications for high-risk patterns such as tools executed outside business hours, repeated authentication failures, unusual parameter values, or sudden spikes in data access. Alerts turn raw telemetry into actionable signals.

**Network telemetry as a corroborating signal**  
NSG Flow Logs and Traffic Analytics capture network-level behavior, including outbound connections, lateral movement, and unexpected traffic patterns. When correlated with application and identity logs, network telemetry helps confirm exfiltration paths and attacker behavior.

**Key Takeaways**:

- Centralize logs in Azure Log Analytics with a minimum 90-day retention
- Enable diagnostic settings on all Azure resources to forward logs automatically
- Instrument MCP servers with OpenTelemetry and MCP-specific context
- Enable NSG Flow Logs and Traffic Analytics for network visibility
- Create alerts for suspicious patterns such as off-hours access, auth failures, an anonymous tool usage

---

## Next Steps

- **Related risks**: All OWASP MCP Top 10 risks benefit from proper telemetry and monitoring
- **Incident response**: Use logs to investigate [MCP01: Token Mismanagement](mcp01-token-mismanagement.md) and [MCP07: Authorization](mcp07-authz.md) incidents
- **Strategic guidance**: [Enterprise Patterns & Lessons Learned](../adoption/enterprise-patterns.md#layered-control-strategy) for comprehensive monitoring strategies
- **Back to**: [OWASP MCP Top 10](../index.md#owasp-mcp-top-10)
