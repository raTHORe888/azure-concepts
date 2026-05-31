# Azure Networking

Detailed notes and workflows for Azure networking design and operations.

## Documentation Safety and Source Policy

- No hardcoded secrets, keys, tokens, passwords, or connection strings are included.
- Examples use placeholders only (for example `<resource-group>`, `<vnet-name>`).
- Explanations and guidance are derived from public Microsoft Azure documentation.
- Do not store real credentials in docs or scripts.

## Topics

1. [VNet, Subnetting, NSG, UDR, DNS, Private Endpoint](01-vnet-subnetting-nsg-udr-dns-private-endpoint.md)
2. [Load Balancer vs Application Gateway](02-load-balancer-vs-application-gateway.md)
3. [Why Network + Identity Together](03-why-network-and-identity-together.md)
4. [End-to-End Network Design Workflow](04-end-to-end-network-design-workflow.md)
5. [Incident Troubleshooting Workflow Diagrams](05-incident-troubleshooting-workflow-diagrams.md)

## Suggested order
- Start with topic 1 and 3 first (core concepts + security model)
- Then topic 2 (traffic entry architecture)
- Then topic 4 (design flow)
- Use topic 5 during real incidents
