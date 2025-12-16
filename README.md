# MCP OWASP Top 10 Security Guidance for Azure

Practical security guidance for implementing Model Context Protocol (MCP) systems on Azure, aligned with the [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/).

## 📖 Documentation

**[View the full guide →](https://microsoft.github.io/mcp-azure-security-guide/)**

This repository contains comprehensive security guidance covering:

- **MCP01**: Token Mismanagement
- **MCP02**: Privilege Escalation
- **MCP03**: Tool Poisoning
- **MCP04**: Supply Chain Vulnerabilities
- **MCP05**: Command Injection
- **MCP06**: Prompt Injection
- **MCP07**: Authorization Failures
- **MCP08**: Insufficient Telemetry
- **MCP09**: Shadow Servers
- **MCP10**: Context Oversharing

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
pip install mkdocs-material

# Serve locally
mkdocs serve

# Build static site
mkdocs build
```

## 🤝 Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
