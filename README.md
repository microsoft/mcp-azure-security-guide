# MCP OWASP Top 10 Security Guidance for Azure

Practical security guidance for implementing Model Context Protocol (MCP) systems on Azure, aligned with the [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/).

## 📖 Documentation

**[View the full guide →](https://microsoft.github.io/mcp-azure-security-guide/)**

## 🚦 Start Here

If you're new to this guide, begin with these pages:

- **[When to Use MCP](https://microsoft.github.io/mcp-azure-security-guide/adoption/when-to-use-mcp/)**
- **[Migration Guidance](https://microsoft.github.io/mcp-azure-security-guide/adoption/migration-guidance/)**
- **[Deployment Patterns](https://microsoft.github.io/mcp-azure-security-guide/adoption/deployment-architecture/)**
- **[OWASP MCP Top 10](https://microsoft.github.io/mcp-azure-security-guide/#owasp-mcp-top-10)**

This repository contains comprehensive security guidance covering:

- **MCP01**: Token Mismanagement and Secret Exposure
- **MCP02**: Privilege Escalation
- **MCP03**: Tool Poisoning
- **MCP04**: Software Supply Chain Attacks & Dependency Tampering
- **MCP05**: Command Injection & Execution
- **MCP06**: Intent Flow Subversion
- **MCP07**: Insufficient Authentication & Authorization
- **MCP08**: Lack of Audit & Telemetry
- **MCP09**: Shadow MCP Servers
- **MCP10**: Context Injection & Over-Sharing

Each guide provides:
- Detailed threat descriptions
- Real-world attack scenarios
- Azure-specific mitigation strategies
- Reference architectures and code examples

## 🏗️ About MCP

The Model Context Protocol (MCP) is a standardized way for AI assistants to connect to tools and data sources. This guide helps you secure these integrations when deploying on Azure.

## 🚀 Local Development

To build and preview the documentation locally:

```bash
# Install dependencies
pip install mkdocs mkdocs-material mkdocs-glightbox

# Serve locally
mkdocs serve

# Build static site
mkdocs build
```

## 🤝 Contributing

Contributions are welcome. Improvements that increase accuracy, consistency, cross-linking, or newcomer clarity are especially useful for this guide.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
