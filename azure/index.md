---
domain: azure
type: index
updated: 2026-10-30
---

# Azure — Domain Index

Reference notes on the Azure services most relevant to the AZ-204 certification (Developing Solutions for Microsoft Azure). Translated from the personal Obsidian `Azure/` folder; each page ends with the exam keys for that service.

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[azure-container-services]] | Look when choosing how to run containers in Azure: purpose and use cases of ACI, App Service, Container Apps and AKS | AZ-204 study notes |
| [[dns-records-custom-domains]] | Look when studying custom domains or SSL in App Service: A, CNAME and TXT records, how to verify a domain and redirect traffic | AZ-204 study notes |
| [[azure-cosmos-db]] | Look when studying Cosmos DB: account → database → container hierarchy, partition key, RU/s and the 5 consistency levels with their internal mechanisms | AZ-204 study notes |
| [[azure-functions]] | Look when studying Azure Functions or serverless: the problem it solves, trigger table, bindings, `func.Out`, `$return`, AuthLevel and why you always need a Storage Account | AZ-204 study notes |
| [[app-service-tiers-and-scaling]] | Look when studying App Service tiers or autoscaling: Shared doesn't autoscale, Standard does (cheapest option), and the scaling CLI commands | AZ-204 study notes |
| [[app-service-deployment-slots]] | Look when studying deployment slots, warm-up or `applicationInitialization` questions: two-phase swap, sticky vs swappable settings, `web.config` warm-up, auto-swap | AZ-204 study notes |
| [[app-service-session-management]] | Look when studying session state, App Service scale-out or ARR questions: shared session (Redis, SQL Server) and output caching with ARR | AZ-204 study notes |
| [[azure-api-management]] | Look when studying API Management or centralized API management: the problem APIM solves, components (Gateway, Developer Portal, policies), request flow and configuration steps | AZ-204 study notes |
| [[azure-event-hubs]] | Look when studying Event Hubs or large-scale event ingestion: partitions, consumer groups, retention and Event Hubs Capture to Blob/Data Lake | AZ-204 study notes |
| [[azure-event-grid]] | Look when studying Event Grid or reacting to events in Azure: publishers, topics, handlers and comparison with Event Hubs | AZ-204 study notes |
| [[azure-service-bus]] | Look when studying asynchronous messaging or service decoupling: queues, topics/subscriptions, DLQ, sessions and receive modes | AZ-204 study notes |
| [[azure-logic-apps]] | Look when studying service integration, workflow orchestration or the differences between messaging services: triggers/actions/connectors, comparison with Functions, Event Grid and Service Bus, Consumption vs Standard | AZ-204 study notes |
| [[azure-application-insights]] | Look when studying Application Insights or app telemetry: SDK, Usage Analytics, Session Id and the exam question on which data not to capture | AZ-204 study notes |
| [[microsoft-entra-id]] | Look when studying authentication, identity or Managed Identities — read before any other security page: Entra ID, Tenant, Service Principals, App Registration, Managed Identity and the Microsoft Identity Platform | AZ-204 study notes |
| [[azure-blob-storage]] | Look when studying object storage or access-cost vs storage-cost questions: the four access tiers (Hot, Cool, Cold, Archive) | AZ-204 study notes |
| [[azure-storage-access-and-permissions]] | Look when studying authentication and permissions in Azure Storage or RBAC and `user_impersonation` questions: the three Blob access scenarios, data vs management roles, Delegated vs Application | AZ-204 study notes |
| [[azure-cdn]] | Look when studying static content delivery or caching rules questions: how PoPs work and the 4 caching behaviors (Bypass, Default, Override, Set if missing) | AZ-204 study notes |
| [[azcopy]] | Look when studying data transfer in Azure Storage or migration questions: authentication (Azure AD vs SAS), `copy` vs `sync`, resumable jobs and server-side copy | AZ-204 study notes |
| [[azure-rbac]] | Look when studying Azure permissions or who-can-do-what questions: identity + role + scope, scope hierarchy, action syntax (`*/read`), built-in roles and data vs management | AZ-204 study notes |
| [[microsoft-graph]] | Look when studying Microsoft Graph or API query parameter questions: OData syntax (`$filter`, `$select`, `eq`), direct REST vs SDK, exam keys | AZ-204 study notes |
| [[azure-app-configuration]] | Look when studying App Configuration or centralized settings and feature flags: key-values, labels per environment, TargetingFilter/PercentageFilter, the Sentinel refresh pattern and Key Vault integration | AZ-204 study notes |
