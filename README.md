# 🏢 TCCC Hub - Azure Deployment Guide

![tccc-multiagent-azure-infra](Deployment/img/Image%20cross.png)


This guide is **exclusively** for deploying the TCCC Hub in The Coca-Cola Company subscription/tenant.

## 📋 Prerequisites

- Active TCCC Azure subscription
- Contributor or Owner permissions in the subscription
- Azure CLI installed and configured
- ARCA information (Tenant ID, App ID) for cross-tenant authentication

## 🚀 Deployment Options

### Option 1: Complete Infrastructure (Recommended for Production)

Includes **ALL** resources needed for a complete production environment.

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-hub-prod --location eastus

# Complete deployment with ALL resources
az deployment group create \
  --resource-group rg-tccc-hub-prod \
  --template-file tccc-hub-infra.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    vnetAddressPrefix="10.1.0.0/16" \
    arcaTenantId="<ARCA_TENANT_ID>" \
    arcaAppId="<ARCA_APP_ID>" \
    enableCrossTenantPeering=true \
    arcaVnetResourceId="<ARCA_VNET_RESOURCE_ID>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account with network ACLs
- ✅ Key Vault
- ✅ Azure AI Foundry (AI Hub for orchestration)
- ✅ Cosmos DB (Database)
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ✅ Configuration for VNET Peering

---

### Option 2: Base Infrastructure (Development/Testing)

Minimal infrastructure **WITHOUT** Cosmos DB or AI Foundry to reduce costs.

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-hub-dev --location eastus

# Deploy without Cosmos DB or AI Foundry
az deployment group create \
  --resource-group rg-tccc-hub-dev \
  --template-file tccc-hub-infra.json \
  --parameters \
    environment=dev \
    deployNetworking=true \
    deployAIFoundry=false \
    deployCosmosDB=false \
    vnetAddressPrefix="10.1.0.0/16" \
    arcaTenantId="<ARCA_TENANT_ID>" \
    arcaAppId="<ARCA_APP_ID>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account
- ✅ Key Vault
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ❌ Azure AI Foundry (not included)
- ❌ Cosmos DB (not included)

**Ideal for:**
- Local development
- Integration testing
- Low-cost environments

---

### Option 3: Custom Deployment

Select exactly which resources you need using parameters.

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-hub-custom --location eastus

# Custom deployment with specific parameters
az deployment group create \
  --resource-group rg-tccc-hub-custom \
  --template-file tccc-hub-infra.json \
  --parameters \
    environment=test \
    deployNetworking=<true|false> \
    deployAIFoundry=<true|false> \
    deployCosmosDB=<true|false> \
    vnetAddressPrefix="10.1.0.0/16" \
    arcaTenantId="<ARCA_TENANT_ID>" \
    arcaAppId="<ARCA_APP_ID>" \
    enableCrossTenantPeering=<true|false> \
    arcaVnetResourceId="<ARCA_VNET_ID_OPTIONAL>"
```

**Configurable parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `environment` | string | dev | Environment: dev, test, prod |
| `deployNetworking` | bool | true | Create VNET and network components |
| `deployAIFoundry` | bool | true | Include Azure AI Foundry |
| `deployCosmosDB` | bool | true | Include Cosmos DB |
| `vnetAddressPrefix` | string | 10.1.0.0/16 | VNET IP range |
| `enableCrossTenantPeering` | bool | false | Enable peering with ARCA |
| `arcaVnetResourceId` | string | "" | ARCA VNET ID (if peering) |

## 📊 Options Comparison

| Resource | Option 1 (Full) | Option 2 (Base) | Option 3 (Custom) |
|----------|-----------------|-----------------|-------------------|
| VNET + NSGs | ✅ | ✅ | Configurable |
| Storage Account | ✅ | ✅ | ✅ Always |
| Key Vault | ✅ | ✅ | ✅ Always |
| Function App | ✅ | ✅ | ✅ Always |
| App Insights | ✅ | ✅ | ✅ Always |
| AI Foundry | ✅ | ❌ | Configurable |
| Cosmos DB | ✅ | ❌ | Configurable |
| VNET Peering | ✅ | Optional | Configurable |
| **Estimated Cost** | $600-800/month | $60-120/month | Variable |

## 🔧 Post-Deployment

### 1. Get deployment information:
```bash
# View deployment outputs
az deployment group show \
  --resource-group rg-tccc-hub-prod \
  --name tccc-hub-infra \
  --query properties.outputs
```

### 2. Configure App Registration (if it doesn't exist):
```bash
# Create App Registration for TCCC Hub
az ad app create \
  --display-name "TCCC-Hub-Orchestrator" \
  --sign-in-audience AzureADMultipleOrgs
```

### 3. Deploy Function App code:
```bash
cd ../../tccc-hub
func azure functionapp publish <FUNCTION_APP_NAME>
```

### 4. Configure VNET Peering (if applicable):
```bash
# Get TCCC VNET ID
TCCC_VNET_ID=$(az network vnet show \
  --resource-group rg-tccc-hub-prod \
  --name tccc-hub-prod-vnet-* \
  --query id -o tsv)

# Share with ARCA team to establish peering
```

## ⚠️ Important Considerations

1. **IP Ranges**: TCCC VNET uses 10.1.0.0/16. DO NOT change if planning peering with ARCA.
2. **Unique names**: The template automatically adds a unique suffix.
3. **Secrets**: Never include secrets in parameters. Use Key Vault post-deployment.
4. **Costs**: Review cost estimation before choosing an option.
5. **Hub Role**: TCCC Hub orchestrates all bottler communications - ensure proper configuration.

## 🆘 Support

- **TCCC Infrastructure Team**: infra@coca-cola.com
- **Technical documentation**: See `TCCC-RESOURCES-DETAIL.md`
- **Network architecture**: See `CROSS-TENANT-VNET-ARCHITECTURE.md`

---
*Last updated: ${new Date().toISOString()}*
