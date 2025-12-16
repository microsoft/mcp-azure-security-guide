# MCP07: Insufficient Authentication & Authorization

### Azure Implementation: FULL

> **Real-World Scenario**: The Wrong Audience
>
> A company runs two MCP servers: one for HR (with access to employee data) and one for Finance (with access to accounting systems). Both use OAuth tokens from the same identity provider. An attacker obtains a valid token intended for the HR server through social engineering. They then present this token to the Finance server. Because the Finance server only checks that the token is valid, and not that it was issued specifically for Finance, the attacker gains unauthorized access to financial data using an HR credential.
>
> **Think of it like**: A concert ticket that only says “VALID TICKET” without specifying which concert. You could use a ticket for last week’s jazz concert to get into tonight’s rock show because nobody checks which event the ticket was for.

## Understanding the Risk

The MCP specification requires OAuth 2.1 with Resource Indicators (RFC 8707). This means tokens must be issued for a specific “audience” (the intended MCP server), and servers must validate that tokens were issued for them. MCP servers are peers, not interchangeable resources. Without proper audience validation, tokens issued for one MCP server can be replayed against another, resulting in unauthorized access across trust boundaries.

## The Azure Solution

![MCP07 Authz](../diagrams/mcp07.png)

**Strong identity boundaries per MCP server**  
Each MCP server must have its own Entra ID App Registration with a unique Application ID URI. Clients must explicitly request tokens for the specific MCP server they intend to call.

**Audience validation at the gateway**  
Azure API Management validates the aud (audience) claim on every request. Tokens issued for the HR MCP server are rejected by the Finance MCP server because the audience does not match.

**Defense-in-depth token validation**  
Audience validation must occur in both APIM (first layer) and within the MCP server code (second layer). If the gateway is misconfigured or bypassed, the server still enforces authorization correctly.

**Protected Resource Metadata**  
MCP servers publish OAuth metadata at /.well-known/oauth-protected-resource (RFC 9728), clearly advertising required audiences and scopes. This ensures clients request correctly scoped tokens and reduces accidental misconfiguration.

**Network isolation as a backstop**  
Even with a valid token, network isolation limits who can reach the MCP server:

- MCP servers have no public IP addresses
- Servers are reachable only through APIM
- NSG rules allow inbound traffic exclusively from the APIM subnet
- Stolen tokens cannot be used directly from the internet

This combines identity validation and network enforcement for true defense-in-depth.

**Key Takeaways**:

- Create a separate Entra ID App Registration for each MCP server
- Configure APIM to validate the ‘aud’ claim matches the specific server
- Deploy MCP servers in private subnets with no public Ips
- Use NSGs to allow inbound traffic only from APIM subnet
- Also validate JWT audience inside your MCP server code (defense-in-depth)
