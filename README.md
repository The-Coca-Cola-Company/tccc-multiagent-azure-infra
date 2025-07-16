# 🏢 TCCC Hub - Azure Deployment Guide

![tccc-multiagent-azure-infra](Deployment/img/image%20cross.png)

> 🚨 **CRITICAL: Two-Stage Deployment Required**  
> You **MUST** deploy in two separate steps to prevent Key Vault access failures and deployment race conditions.

---

## 🔄 Two-Stage Deployment Process

### Why Two Stages?
The Function App requires access to Key Vault secrets during deployment. A single-stage deployment creates a race condition where the Function App tries to access secrets before permissions are properly configured, causing this error:

> ❌ **AccessToKeyVaultDenied** – Unable to resolve setting: WEBSITE_CONTENTAZUREFILECONNECTIONSTRING

---

## Step 1: 🔐 Pre-Configuration Setup

This deploys foundational infrastructure and configures permissions:

**Includes:**
- ✅ Key Vault + managed identity permissions
- ✅ Storage Account + connection strings 
- ✅ Application Insights + secrets
- ✅ App Service Plan (EP1 for VNet, Y1 for basic)
- ✅ VNet, NSGs, and networking (if enabled)
- ✅ Azure AI Foundry (if enabled)
- ✅ Cosmos DB (if enabled)

### 1️⃣ Full Production Pre-Config
[![Deploy Pre-Config to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-multiagent-azure-infra%2Fmain%2FDeployment%2Ftccc-hub-preconfig.json)

### 2️⃣ Minimal Dev/Test Pre-Config
[![Deploy Pre-Config Minimal](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-multiagent-azure-infra%2Fmain%2FDeployment%2Ftccc-hub-preconfig.json)

*Use same template - disable AI Foundry/Cosmos DB in parameters*

---

### ⏳ **Wait 60-90 seconds** before proceeding to Step 2

This allows Azure to:
- Complete role assignments
- Propagate permissions
- Initialize Key Vault access

---

## Step 2: 🚀 TCCC Hub Function Deployment

This deploys the Function App using the infrastructure from Step 1:

**Includes:**
- ✅ Function App with system-assigned identity
- ✅ VNet integration (if networking enabled)
- ✅ Key Vault references for app settings
- ✅ Role assignments for storage and Key Vault access

### 1️⃣ Full Production Function App
[![Deploy Function App](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-multiagent-azure-infra%2Fmain%2FDeployment%2Ftccc-hub-main.json)

### 2️⃣ Minimal Dev/Test Function App
[![Deploy Function App Minimal](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-multiagent-azure-infra%2Fmain%2FDeployment%2Ftccc-hub-main.json)

**📋 Required Parameters for Step 2:**
- `preDeployedStorageAccountName`: From Step 1 outputs
- `preDeployedKeyVaultName`: From Step 1 outputs  
- `preDeployedAppServicePlanName`: From Step 1 outputs
- `preDeployedVnetName`: From Step 1 outputs (if networking enabled)

---

## 📋 Prerequisites

- Active TCCC Azure subscription
- Contributor or Owner permissions in the subscription
- Azure CLI installed and configured
- Bottler information (Tenant IDs, App IDs) for cross-tenant authentication

---

## 🎯 CLI Deployment (Advanced)

### Option 1: Complete Production Deployment

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-hub-prod --location eastus

# STEP 1: Pre-Configuration
az deployment group create \
  --resource-group rg-tccc-hub-prod \
  --template-file tccc-hub-preconfig.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    vnetAddressPrefix="10.1.0.0/16" \
    bottlerTenantIds='["<BOTTLER_TENANT_ID_1>","<BOTTLER_TENANT_ID_2>"]' \
    bottlerAppIds='["<BOTTLER_APP_ID_1>","<BOTTLER_APP_ID_2>"]' \
    enableCrossTenantPeering=true

# Get outputs from Step 1
STORAGE_NAME=$(az deployment group show --resource-group rg-tccc-hub-prod --name tccc-hub-preconfig --query properties.outputs.storageAccountName.value -o tsv)
KV_NAME=$(az deployment group show --resource-group rg-tccc-hub-prod --name tccc-hub-preconfig --query properties.outputs.keyVaultName.value -o tsv)
ASP_NAME=$(az deployment group show --resource-group rg-tccc-hub-prod --name tccc-hub-preconfig --query properties.outputs.appServicePlanName.value -o tsv)
VNET_NAME=$(az deployment group show --resource-group rg-tccc-hub-prod --name tccc-hub-preconfig --query properties.outputs.vnetName.value -o tsv)

# Add required secrets to Key Vault
az keyvault secret set --vault-name $KV_NAME --name storage-connection-string --value "$(az storage account show-connection-string --name $STORAGE_NAME --resource-group rg-tccc-hub-prod --query connectionString -o tsv)"

# Wait 60 seconds for permissions to propagate
sleep 60

# STEP 2: Function App Deployment
az deployment group create \
  --resource-group rg-tccc-hub-prod \
  --template-file tccc-hub-main.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    bottlerTenantIds='["<BOTTLER_TENANT_ID_1>","<BOTTLER_TENANT_ID_2>"]' \
    bottlerAppIds='["<BOTTLER_APP_ID_1>","<BOTTLER_APP_ID_2>"]' \
    preDeployedStorageAccountName=$STORAGE_NAME \
    preDeployedKeyVaultName=$KV_NAME \
    preDeployedAppServicePlanName=$ASP_NAME \
    preDeployedVnetName=$VNET_NAME
```

### Option 2: Development/Testing Deployment

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-hub-dev --location eastus

# STEP 1: Pre-Configuration (minimal)
az deployment group create \
  --resource-group rg-tccc-hub-dev \
  --template-file tccc-hub-preconfig.json \
  --parameters \
    environment=dev \
    deployNetworking=true \
    deployAIFoundry=false \
    deployCosmosDB=false \
    vnetAddressPrefix="10.1.0.0/16" \
    bottlerTenantIds='["<BOTTLER_TENANT_ID>"]' \
    bottlerAppIds='["<BOTTLER_APP_ID>"]'

# Get outputs and add secrets (same as above)
# Wait 60 seconds
# Deploy Step 2 with same parameters
```

---

## 📊 Deployment Comparison

 < /dev/null |  Resource | Step 1 (Pre-Config) | Step 2 (Function App) |
|----------|---------------------|----------------------|
| Storage Account | ✅ | References Step 1 |
| Key Vault | ✅ | References Step 1 |
| App Service Plan | ✅ | References Step 1 |
| VNet + NSGs | ✅ | References Step 1 |
| AI Foundry | ✅ | References Step 1 |
| Cosmos DB | ✅ | References Step 1 |
| Function App | ❌ | ✅ Deployed |
| Role Assignments | ❌ | ✅ Deployed |

---

## 🔧 Post-Deployment Steps

### 1. Get deployment information:
```bash
# View Step 2 outputs
az deployment group show \
  --resource-group rg-tccc-hub-prod \
  --name tccc-hub-main \
  --query properties.outputs
```

### 2. Deploy Function App code:
```bash
cd ../../tccc-hub
func azure functionapp publish <FUNCTION_APP_NAME_FROM_OUTPUT>
```

### 3. Test deployment:
```bash
# Test Function App endpoint
curl https://<FUNCTION_APP_NAME>.azurewebsites.net/api/health
```

### 4. Configure cross-tenant peering:
```bash
# Get TCCC VNET ID
TCCC_VNET_ID=$(az network vnet show \
  --resource-group rg-tccc-hub-prod \
  --name $VNET_NAME \
  --query id -o tsv)

# Share with bottler teams for peering setup
echo "TCCC VNET ID for peering: $TCCC_VNET_ID"
```

---

## ⚠️ Important Notes

1. **Mandatory Two-Stage Process**: Never skip Step 1 or combine steps
2. **Wait Time**: Always wait 60-90 seconds between stages
3. **Parameter Consistency**: Use same parameters for both stages
4. **Secret Management**: Step 1 creates secrets, Step 2 consumes them
5. **IP Ranges**: TCCC VNET uses `10.1.0.0/16` - don't change for bottler peering
6. **Hub Role**: TCCC Hub orchestrates all bottler communications

---

## 🆘 Troubleshooting

### Common Issues:

**❌ AccessToKeyVaultDenied**
- **Cause**: Deployed in single stage or insufficient wait time
- **Solution**: Redeploy using two-stage process with proper wait

**❌ Function App deployment fails**
- **Cause**: Missing Step 1 outputs or incorrect parameter names
- **Solution**: Verify Step 1 completed and use exact output names

**❌ VNet integration fails**
- **Cause**: Y1 SKU used with networking enabled
- **Solution**: Templates auto-select EP1 SKU when networking=true

**❌ Cross-tenant authentication fails**
- **Cause**: Missing or incorrect bottler tenant/app IDs
- **Solution**: Verify bottler credentials and update parameters

---

## 🆘 Support

- **TCCC Infrastructure Team**: Emerging Technology
- **Technical documentation**: [TCCC-RESOURCES-DETAIL.md](Docs/TCCC-RESOURCES-DETAIL.md)
- **Network architecture**: [CROSS-TENANT-VNET-ARCHITECTURE.md](Docs/CROSS-TENANT-VNET-ARCHITECTURE.md)
- **Security guide**: [CROSS-TENANT-AUTHENTICATION-GUIDE.md](Docs/CROSS-TENANT-AUTHENTICATION-GUIDE.md)

---

*Last updated: 2025-07-16*
*Critical deployment fix: Two-stage process to prevent Key Vault access failures*
