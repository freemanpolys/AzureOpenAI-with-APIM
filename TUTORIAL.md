# Tutorial: Creating Azure API Management with Two Azure OpenAI Backends

This tutorial will guide you through creating an Azure API Management (APIM) instance configured with two Azure OpenAI backends for load balancing and failover resiliency.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Your Specific Setup](#your-specific-setup)
- [Architecture](#architecture)
- [Step-by-Step Setup](#step-by-step-setup)
- [Testing Your Configuration](#testing-your-configuration)
- [Advanced Configurations](#advanced-configurations)
- [Troubleshooting](#troubleshooting)

## Overview

This solution enables you to:

- **Centralize access** to Azure OpenAI services through a single APIM endpoint
- **Implement resiliency** with automatic failover between two Azure OpenAI instances
- **Manage access** with subscription keys instead of sharing Azure OpenAI keys
- **Monitor usage** with Application Insights and Log Analytics
- **Control costs** with rate limiting and token tracking

## Your Specific Setup

This tutorial is customized for your Azure AI Foundry deployments:

| Component | Primary Backend | Secondary Backend |
|-----------|----------------|-------------------|
| **AI Foundry Name** | doveia-frc-ai-foundry | doveia-sec-ai-foundry |
| **Region** | France Central | Sweden Central |
| **API Base URL** | https://doveia-frc-ai-foundry.openai.azure.com/openai | https://doveia-sec-ai-foundry-resource.openai.azure.com/openai |
| **Deployment** | gpt-4o-mini | gpt-4o-mini |

**Important Notes:**
- Both AI Foundry projects must have the **same deployment name** (gpt-4o-mini) for load balancing to work correctly
- The backend URLs in APIM should use the base `/openai` path (without `/v1`) as Azure OpenAI standard
- Your full API URLs include `/v1/` but the backend URL in APIM should use `/openai`
- APIM will route requests between both backends with automatic failover on HTTP 429 or 500+ errors

## Prerequisites

Before you begin, ensure you have:

### Azure Resources

1. **Azure Subscription** with Contributor permissions
2. **Two Azure AI Foundry Projects** (already deployed):
   - Primary: `doveia-frc-ai-foundry` in France Central
   - Secondary: `doveia-sec-ai-foundry` in Sweden Central
3. **Model deployed** in both AI Foundry projects:
   - Deployment name: `gpt-4o-mini` (must be identical in both)
4. **Resource Group** (or ability to create one for APIM)

### Information You Need

Here's what you'll use in the configuration (already gathered above):

| Parameter | Your Value |
|-----------|-----------|
| Primary Service Name | `doveia-frc-ai-foundry` |
| Primary Resource Group | _(your AI Foundry RG)_ |
| Secondary Service Name | `doveia-sec-ai-foundry-resource` |
| Secondary Resource Group | _(your AI Foundry RG)_ |
| Deployment Name | `gpt-4o-mini` |
| API Version | `2024-02-15-preview` or `2024-08-01-preview` |

### Tools Required

- **Azure Portal** access with appropriate permissions to create and configure resources

## Architecture

### What You'll Create

This solution involves creating and configuring:

1. **API Management Service** - Central gateway for Azure OpenAI access
2. **Application Insights** (optional) - Request monitoring and analytics
3. **Log Analytics Workspace** (optional) - Centralized logging and queries
4. **Two Backend Configurations**:
   - Primary Azure OpenAI endpoint
   - Secondary Azure OpenAI endpoint (failover)

### How It Works

```
Client Request
    |
    v
[API Management]
    |
    v
[OpenAI-LB Load Balancer]
    |
    +---> [Primary Azure OpenAI] (France Central)
    |           |
    |           +---> Success --> Return Response
    |
    +---> [Secondary Azure OpenAI] (Sweden Central)
                |
                +---> Success --> Return Response
```

**Key Features:**

- **Load Balancer Pool** - Automatic round-robin distribution between backends
- **Managed Identity Authentication** - Secure connection without storing keys
- **Automatic Failover** - If one backend fails, traffic routes to the healthy backend
- **Bearer Token Support** - Use Azure OpenAI SDKs without code changes (supports both `Authorization: Bearer` and `Ocp-Apim-Subscription-Key`)
- **Token Tracking** - Monitor usage per subscription for cost management
- **Rate Limiting** - Control usage and costs per subscription

## Step-by-Step Setup

### Step 1: Verify Your Azure AI Foundry Setup

Since you already have your Azure AI Foundry projects deployed, let's verify everything is ready:

#### 1.1 Verify Both AI Foundry Projects Exist

1. Navigate to **Azure Portal** > Search for "Azure AI Studio" or "AI Foundry"
2. Confirm you can see:
   - **doveia-frc-ai-foundry** (France Central)
   - **doveia-sec-ai-foundry** (Sweden Central)

#### 1.2 Verify Deployment Names Match

This is critical for load balancing to work:

1. Go to **doveia-frc-ai-foundry** > **Deployments**
2. Confirm you have a deployment named **gpt-4o-mini**
3. Note the deployment name exactly as shown

4. Go to **doveia-sec-ai-foundry** > **Deployments**
5. Confirm the deployment name is **exactly the same**: **gpt-4o-mini**

If the deployment names don't match, either:
- Rename one to match the other, OR
- Deploy a new model with matching names in both projects

#### 1.3 Collect Resource Group Names

You'll need the resource group name for each AI Foundry project:

1. Go to **doveia-frc-ai-foundry** > **Overview**
2. Note the **Resource group** name (e.g., `rg-ai-foundry-france`)

3. Go to **doveia-sec-ai-foundry** > **Overview**
4. Note the **Resource group** name (e.g., `rg-ai-foundry-sweden`)

**Write these down - you'll need them for the configuration:**
- Primary RG: `___________________`
- Secondary RG: `___________________`

### Step 2: Create API Management Service

#### 2.1 Create APIM Instance

1. Navigate to **Azure Portal** (portal.azure.com)
2. Click **+ Create a resource**
3. Search for **API Management** and select it
4. Click **Create**

5. Fill in the **Basics** tab:
   | Field | Value |
   |-------|-------|
   | Subscription | Your subscription |
   | Resource Group | Create new or select existing (e.g., `rg-apim-loadbalancer`) |
   | Region | East US (or your preferred region) |
   | Resource name | Choose a unique name (e.g., `apim-doveia-openai`) |
   | Organization name | `Doveia` |
   | Administrator email | `admin@doveia.com` |
   | Pricing tier | **Developer** (no SLA) or **Basic/Standard/Premium** (with SLA) |

6. Click **Review + create** > **Create**

7. **Wait 30-45 minutes** - APIM deployment takes time

#### 2.2 Enable Managed Identity

Once APIM is deployed:

1. Go to your APIM service in Azure Portal
2. Navigate to **Settings** > **Managed identities**
3. Under **System assigned** tab, toggle **Status** to **On**
4. Click **Save**
5. Note the **Object (principal) ID** - you'll need this for permissions

#### 2.3 Create Application Insights (Optional but Recommended)

1. In Azure Portal, click **+ Create a resource**
2. Search for **Application Insights** and select it
3. Click **Create**
4. Fill in details:
   - Resource Group: Same as APIM
   - Name: `appinsights-apim-openai`
   - Region: Same as APIM
   - Resource Mode: **Workspace-based**
   - Log Analytics Workspace: Create new or select existing
5. Click **Review + create** > **Create**

### Step 3: Configure API Management Backends

#### 3.1 Create Primary Backend

1. Go to your APIM service > **Backends** (under APIs section)
2. Click **+ Add**
3. Fill in backend details:
   | Field | Value |
   |-------|-------|
   | Name | `aoai-primary-backend` |
   | Type | **HTTP(s) endpoint** |
   | Runtime URL | `https://doveia-frc-ai-foundry.openai.azure.com/openai` |
   | Description | Primary Azure OpenAI backend (France Central) |
   | Validate certificate chain | ✅ Checked |
   | Validate certificate name | ✅ Checked |

4. Under **Credentials** section:
   - Select **Authentication** type: **System-assigned managed identity**
   - In the **Resource** field, enter: `https://cognitiveservices.azure.com`

     **Note:** This is the Azure Cognitive Services OAuth resource URI, not your AI Foundry name. This tells Azure which service the managed identity should authenticate to.

   - Authority: Leave as default (https://login.microsoftonline.com)

5. Click **Create**

#### 3.2 Create Secondary Backend

1. In APIM > **Backends**, click **+ Add** again
2. Fill in backend details:
   | Field | Value |
   |-------|-------|
   | Name | `aoai-secondary-backend` |
   | Type | **HTTP(s) endpoint** |
   | Runtime URL | `https://doveia-sec-ai-foundry-resource.openai.azure.com/openai` |
   | Description | Secondary Azure OpenAI backend (Sweden Central) |
   | Validate certificate chain | ✅ Checked |
   | Validate certificate name | ✅ Checked |

3. Under **Credentials** section:
   - Select **Authentication** type: **System-assigned managed identity**
   - In the **Resource** field, enter: `https://cognitiveservices.azure.com`

     **Note:** Use the same Cognitive Services OAuth URI for both backends.

   - Authority: Leave as default (https://login.microsoftonline.com)

4. Click **Create**

#### 3.3 Create Load Balancer Backend Pool

Now create a load balancer pool to distribute traffic between both backends:

1. In APIM > **Backends**, click **+ Add**
2. Select **Type**: **Load Balanced Pool**
3. Fill in details:
   | Field | Value |
   |-------|-------|
   | Name | `OpenAI-LB` |
   | Description | Load balancer for Azure OpenAI backends |
   | Load balancing algorithm | **Round Robin** (or **Weighted** if you want priority) |

4. Under **Backends in pool** section:
   - Click **+ Add**
   - Select **aoai-primary-backend**
   - Priority: 1 (or weight if using weighted algorithm)
   - Click **Add**

5. Click **+ Add** again:
   - Select **aoai-secondary-backend**
   - Priority: 2 (or weight if using weighted algorithm)
   - Click **Add**

6. Click **Create**

**What this does:**
- The load balancer will automatically distribute requests between both backends
- If one backend is unavailable or returns errors, traffic automatically fails over to the other
- You can configure priority/weight to prefer one backend over another

### Step 4: Grant APIM Access to Azure OpenAI Services

You need to grant your APIM managed identity permission to access both Azure OpenAI services.

#### 4.1 Grant Access to Primary AI Foundry

1. Navigate to **doveia-frc-ai-foundry** (your primary AI Foundry resource)
2. Click **Access control (IAM)**
3. Click **+ Add** > **Add role assignment**
4. In the **Role** tab:
   - Search for and select **Cognitive Services OpenAI User**
   - Click **Next**
5. In the **Members** tab:
   - Select **Managed identity**
   - Click **+ Select members**
   - Filter by **API Management service**
   - Select your APIM instance (e.g., `apim-doveia-openai`)
   - Click **Select**
6. Click **Review + assign** > **Review + assign**

#### 4.2 Grant Access to Secondary AI Foundry

Repeat the same steps for the secondary AI Foundry:

1. Navigate to **doveia-sec-ai-foundry-resource** (your secondary AI Foundry resource)
2. Click **Access control (IAM)**
3. Click **+ Add** > **Add role assignment**
4. Select **Cognitive Services OpenAI User** role
5. Add the same APIM managed identity as a member
6. Click **Review + assign**

**Verification:** After adding both role assignments, wait 2-3 minutes for permissions to propagate.

### Step 5: Create Azure OpenAI API in APIM

#### 5.1 Create Azure OpenAI API

Since the OpenAPI import requires a specific base URL, we'll create the API manually instead:

1. Go to your APIM service > **APIs**
2. Click **+ Add API**
3. Select **HTTP** (manual API)
4. Fill in API details:
   | Field | Value |
   |-------|-------|
   | Display name | `Azure OpenAI Service API` |
   | Name | `azure-openai-service-api` |
   | Web service URL | `https://doveia-frc-ai-foundry.openai.azure.com/openai` (temporary - will be overridden by backend) |
   | API URL suffix | Leave empty |
   | URL scheme | **HTTPS only** |

5. Click **Create**

#### 5.2 Add Chat Completions Operation

Now add the main operation you'll use:

1. Click on the **Azure OpenAI Service API** you just created
2. Click **+ Add operation**
3. Fill in operation details:
   | Field | Value |
   |-------|-------|
   | Display name | `Creates a completion for the chat message` |
   | Name | `Creates_a_completion_for_the_chat_message` |
   | URL - Method | **POST** |
   | URL - Template | `/deployments/{deployment-id}/chat/completions` |

4. Click the **Template** tab
5. Add a template parameter:
   - **Name**: `deployment-id`
   - **Type**: string
   - **Required**: Yes
   - **Description**: Deployment id of the model

6. Click the **Request** tab
7. Click **+ Add query parameter**:
   - **Name**: `api-version`
   - **Type**: string
   - **Required**: Yes
   - **Values**: `2024-02-15-preview` (you can add more versions as needed)

8. Click **Save**

#### 5.3 Configure API to Use Load Balancer Backend and Bearer Token Authentication

To allow Azure OpenAI SDK clients to work seamlessly with APIM using `Authorization: Bearer` tokens:

1. In **APIs**, click on **Azure OpenAI Service API**
2. Click **All operations**
3. In the **Inbound processing** section, click **</>** (Code view)
4. Replace the entire `<inbound>` section with this policy:

```xml
<inbound>
    <base />
    <!-- Extract Bearer token from Authorization header and use it as subscription key -->
    <choose>
        <when condition="@(context.Request.Headers.GetValueOrDefault("Authorization","").StartsWith("Bearer "))">
            <set-header name="Ocp-Apim-Subscription-Key" exists-action="override">
                <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").Substring("Bearer ".Length))</value>
            </set-header>
        </when>
    </choose>
    <!-- Remove Authorization header before forwarding to backend -->
    <set-header name="Authorization" exists-action="delete" />
    <!-- Set backend and authenticate with managed identity -->
    <set-backend-service backend-id="OpenAI-LB" />
    <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
</inbound>
```

5. Click **Save**

**What this does:**
- Accepts `Authorization: Bearer <subscription-key>` header (standard Azure OpenAI SDK format)
- Converts it to APIM's `Ocp-Apim-Subscription-Key` internally
- Removes the Authorization header before forwarding to backend (prevents conflicts)
- Routes all requests to the `OpenAI-LB` load balancer
- Uses managed identity for backend authentication
- Your Azure OpenAI SDK clients work without any code changes!

### Step 6: Configure Circuit Breaker (Optional but Recommended)

Since you're using the `OpenAI-LB` load balancer, it already handles failover automatically. However, you can add a circuit breaker policy for additional resilience:

1. In **Azure OpenAI Service API** > **All operations** > **Policies**
2. Click **</>** (Code view)
3. Update the policy XML to include circuit breaker:

```xml
<policies>
    <inbound>
        <base />
        <set-backend-service backend-id="OpenAI-LB" />
        <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

4. Click **Save**

**What this does:**
- All requests go to the `OpenAI-LB` load balancer
- The load balancer automatically distributes traffic using round-robin or weighted algorithm
- If one backend fails, the load balancer automatically routes to the healthy backend
- No manual retry logic needed - the load balancer handles it

**Optional - Add Retry Policy for Extra Resilience:**

If you want additional retry logic on top of the load balancer, you can add this to the `<backend>` section:

```xml
<backend>
    <retry condition="@(context.Response.StatusCode == 429 || context.Response.StatusCode >= 500)" count="3" interval="1" delta="1" first-fast-retry="true">
        <forward-request buffer-request-body="true" />
    </retry>
</backend>
```

This will retry failed requests 3 times, with the load balancer choosing a different backend each time.

### Step 7: Create Subscription for API Access

1. Go to your APIM service > **Subscriptions**
2. Click **+ Add subscription**
3. Fill in details:
   | Field | Value |
   |-------|-------|
   | Name | `azure-openai-consumer` |
   | Display name | `Azure OpenAI Consumer` |
   | Scope | **API** |
   | API | Select **Azure OpenAI Service API** |
   | Allow tracing | ✅ Checked (for testing) |
4. Click **Create**
5. Click **...** next to the new subscription > **Show/hide keys**
6. Copy the **Primary key** - you'll need this for testing

### Step 8: Connect Application Insights (Optional)

If you created Application Insights in Step 2.3:

1. Go to your APIM service > **Application Insights** (under Monitoring)
2. Click **+ Add**
3. Select your Application Insights instance
4. Set **Sampling** to **100%** (for testing)
5. Check **Always log errors**
6. Click **Create**

### Step 9: Verify Configuration

Let's verify everything is configured correctly:

#### 9.1 Verify Resources Created

1. Navigate to **Azure Portal** > **Resource Groups** > Your APIM resource group
2. Verify these resources exist:
   - API Management service
   - Application Insights (if you created it)
   - Log Analytics workspace (if created with App Insights)

#### 9.2 Verify APIM Configuration

1. **Check API:**
   - Go to your APIM service > **APIs**
   - Verify **Azure OpenAI Service API** is listed

2. **Check Backends:**
   - Go to **Backends**
   - Verify three backends exist:
     - `aoai-primary-backend` (HTTP endpoint to https://doveia-frc-ai-foundry.openai.azure.com/openai)
     - `aoai-secondary-backend` (HTTP endpoint to https://doveia-sec-ai-foundry-resource.openai.azure.com/openai)
     - `OpenAI-LB` (Load Balanced Pool containing both backends)

3. **Check Subscription:**
   - Go to **Subscriptions**
   - Verify your subscription is listed and has keys available

#### 9.3 Verify Managed Identity Permissions

To verify the APIM managed identity has proper access:

1. Go to **Azure Portal** > **doveia-frc-ai-foundry** (your primary AI Foundry)
2. Click **Access control (IAM)**
3. Click **Role assignments**
4. Search for your APIM service name
5. Verify it has **Cognitive Services OpenAI User** role

6. Repeat for **doveia-sec-ai-foundry-resource** (your secondary AI Foundry)

If the role assignments are missing, go back to Step 4 and add them.

## Testing Your Configuration

### Get Your APIM Endpoint URL

1. Navigate to **Azure Portal** > Your APIM service > **Overview**
2. Copy the **Gateway URL** (e.g., `https://apim-doveia-openai.azure-api.net`)

### Get Your Subscription Key

1. In your APIM service, go to **Subscriptions**
2. Find your subscription (e.g., "Azure OpenAI Consumer")
3. Click **...** > **Show/hide keys**
4. Copy the **Primary key**

### Test with PowerShell

**Option 1: Using Bearer Token (Azure OpenAI SDK compatible)**

```powershell
# Set your values
$apimUrl = "https://apim-xxxxx.azure-api.net"  # Your APIM Gateway URL
$deploymentName = "gpt-4o-mini"  # Your deployment name
$apiVersion = "2024-02-15-preview"
$subscriptionKey = "your-subscription-key"  # From Step 7

# Construct URL
$url = "$apimUrl/deployments/$deploymentName/chat/completions?api-version=$apiVersion"

# Headers - Using Bearer token like Azure OpenAI SDK
$headers = @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer $subscriptionKey"
}

# Request body
$body = @{
    messages = @(
        @{
            role = "system"
            content = "You are a helpful AI assistant."
        },
        @{
            role = "user"
            content = "What is Azure API Management?"
        }
    )
    temperature = 0.7
    max_tokens = 800
} | ConvertTo-Json

# Make request
$response = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body

# Display response
$response.choices.message.content
```

**Option 2: Using Ocp-Apim-Subscription-Key (APIM native)**

```powershell
# Headers - Using APIM subscription key header
$headers = @{
    "Content-Type" = "application/json"
    "Ocp-Apim-Subscription-Key" = $subscriptionKey
}

# Make request (same URL and body as above)
$response = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body
$response.choices.message.content
```

### Test with cURL (Bash)

**Option 1: Using Bearer Token (Azure OpenAI SDK compatible)**

```bash
#!/bin/bash

apimUrl="https://apim-xxxxx.azure-api.net"
deploymentName="gpt-4o-mini"
apiVersion="2024-02-15-preview"
subscriptionKey="your-subscription-key"

url="${apimUrl}/deployments/${deploymentName}/chat/completions?api-version=${apiVersion}"

curl "${url}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${subscriptionKey}" \
  -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful AI assistant."
      },
      {
        "role": "user",
        "content": "What is Azure API Management?"
      }
    ],
    "temperature": 0.7,
    "max_tokens": 800
  }'
```

**Option 2: Using Ocp-Apim-Subscription-Key (APIM native)**

```bash
curl "${url}" \
  -H "Content-Type: application/json" \
  -H "Ocp-Apim-Subscription-Key: ${subscriptionKey}" \
  -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful AI assistant."
      },
      {
        "role": "user",
        "content": "What is Azure API Management?"
      }
    ],
    "temperature": 0.7,
    "max_tokens": 800
  }'
```

### Test with Azure OpenAI SDK

The Bearer token support allows you to use official Azure OpenAI SDKs with your APIM endpoint:

**Python SDK:**

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="your-subscription-key",  # Your APIM subscription key
    api_version="2024-02-15-preview",
    azure_endpoint="https://apim-xxxxx.azure-api.net"  # Your APIM Gateway URL
)

response = client.chat.completions.create(
    model="gpt-4o-mini",  # Your deployment name
    messages=[
        {"role": "system", "content": "You are a helpful AI assistant."},
        {"role": "user", "content": "What is Azure API Management?"}
    ],
    temperature=0.7,
    max_tokens=800
)

print(response.choices[0].message.content)
```

**.NET SDK:**

```csharp
using Azure;
using Azure.AI.OpenAI;

var endpoint = new Uri("https://apim-xxxxx.azure-api.net");
var credentials = new AzureKeyCredential("your-subscription-key");
var client = new AzureOpenAIClient(endpoint, credentials);

var chatClient = client.GetChatClient("gpt-4o-mini");

var completion = await chatClient.CompleteChatAsync(
[
    new SystemChatMessage("You are a helpful AI assistant."),
    new UserChatMessage("What is Azure API Management?")
]);

Console.WriteLine(completion.Value.Content[0].Text);
```

**JavaScript/TypeScript SDK:**

```typescript
import { AzureOpenAI } from "openai";

const client = new AzureOpenAI({
  apiKey: "your-subscription-key",
  apiVersion: "2024-02-15-preview",
  endpoint: "https://apim-xxxxx.azure-api.net"
});

const result = await client.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [
    { role: "system", content: "You are a helpful AI assistant." },
    { role: "user", content: "What is Azure API Management?" }
  ],
  temperature: 0.7,
  max_tokens: 800
});

console.log(result.choices[0].message.content);
```

**Benefits:**
- No code changes needed - just update the endpoint and API key
- All SDK features work (streaming, function calling, etc.)
- Load balancing and failover handled transparently by APIM
- Easy migration from direct Azure OpenAI to APIM

### Test Load Balancing and Failover Behavior

The `OpenAI-LB` load balancer will automatically distribute requests between both backends.

#### Test Normal Load Balancing

1. **Make multiple requests** (at least 10) using the scripts above
2. **Check Application Insights:**
   - Go to your APIM service > **Application Insights** > **Transaction search**
   - Filter by "azure-openai-service-api"
   - Look at the backend URLs in the requests
   - With round-robin, you should see requests distributed between both backends:
     - Some to `doveia-frc-ai-foundry.openai.azure.com` (France Central)
     - Some to `doveia-sec-ai-foundry-resource.openai.azure.com` (Sweden Central)

#### Test Failover to Secondary

To test that automatic failover works:

1. **Temporarily remove primary backend from pool:**
   - Go to APIM > **Backends** > `OpenAI-LB`
   - Click **Edit**
   - Remove `aoai-primary-backend` from the pool (or disable it)
   - Save changes

2. **Make test requests** using the scripts above
   - All requests should go to the secondary backend
   - You should still get successful responses

3. **Check Application Insights:**
   - Go to **Application Insights** > **Transaction search**
   - All requests should now show the secondary backend URL

4. **Restore the primary backend:**
   - Go back to **Backends** > `OpenAI-LB`
   - Click **Edit**
   - Add `aoai-primary-backend` back to the pool
   - Save changes
   - Requests will now be distributed across both backends again

## Advanced Configurations

### Adding Rate Limiting

Limit token usage per subscription to control costs:

1. Go to **APIM** > **APIs** > **azure-openai-service-api**
2. Click **All operations** > **Policies** > **Inbound processing**
3. Add this policy:

```xml
<azure-openai-token-limit
    tokens-per-minute="10000"
    counter-key="@(context.Subscription.Id)"
    estimate-prompt-tokens="true"
    tokens-consumed-header-name="consumed-tokens"
    remaining-tokens-header-name="remaining-tokens" />
```

### Adding Token Tracking for Chargeback

Track token usage per subscription for cost allocation:

1. Add this policy to track tokens:

```xml
<azure-openai-emit-token-metric>
    <dimension name="API ID" />
    <dimension name="Subscription ID" />
    <dimension name="User ID" />
    <dimension name="Product ID" />
    <dimension name="Deployment ID" value="@(context.Variables.GetValueOrDefault<string>("deploymentId", "unknown"))" />
</azure-openai-emit-token-metric>
```

2. Query usage in Log Analytics:

```kql
customMetrics
| where name in ("Prompt Tokens", "Completion Tokens")
| extend subscriptionId = tostring(customDimensions['Subscription ID'])
| extend deploymentId = tostring(customDimensions['Deployment ID'])
| extend tokens = toreal(value)
| summarize totalTokens = sum(tokens) by subscriptionId, deploymentId
| order by totalTokens desc
```

### Priority-Based Routing

Route requests to different backends based on priority (useful if you have a PTU deployment for high-priority requests):

```xml
<set-variable name="priorityValue" value="@(context.Request.Headers.GetValueOrDefault("Priority", "0"))" />
<choose>
    <when condition="@(int.Parse((string)context.Variables["priorityValue"]) <= 100)">
        <set-backend-service backend-id="aoai-ptu-backend" />
    </when>
    <otherwise>
        <set-backend-service backend-id="OpenAI-LB" />
    </otherwise>
</choose>
```

**Note:** This routes high-priority requests to a dedicated PTU backend, while normal requests use the load-balanced pool.

Clients include priority in headers:

```bash
curl "${url}" \
  -H "Priority: 50" \
  -H "Ocp-Apim-Subscription-Key: ${subscriptionKey}" \
  -d '{ ... }'
```

### Creating Additional Subscriptions

Create subscriptions for different teams:

1. Go to **APIM** > **Subscriptions** > **+ Add subscription**
2. Fill in details:
   - **Display name**: Team-Marketing
   - **Scope**: API (select azure-openai-service-api)
   - **User**: Select or create user
3. Click **Save**
4. Share the subscription key with the team

### Monitoring and Alerts

Set up monitoring in Application Insights:

1. Go to **Application Insights** > **Logs**
2. Query request metrics:

```kql
requests
| where url contains "azure-openai-service-api"
| summarize count(), avg(duration) by bin(timestamp, 1h), resultCode
| render timechart
```

3. Create alerts:
   - Go to **Alerts** > **+ New alert rule**
   - Set conditions (e.g., failure rate > 5%)
   - Configure action groups for notifications

## Troubleshooting

### Issue: HTTP 401 Unauthorized

**Cause:** APIM managed identity lacks permissions

**Solution:**
1. Go to each Azure OpenAI service > **Access control (IAM)**
2. Click **+ Add** > **Add role assignment**
3. Select **Cognitive Services OpenAI User**
4. Click **Members** > **Managed identity**
5. Select your APIM service
6. Click **Review + assign**

### Issue: HTTP 429 Too Many Requests

**Cause:** Quota exceeded on both backends OR rate limit policy

**Solution:**
1. Check Azure OpenAI quota in Azure Portal
2. Check APIM rate limiting policies
3. Consider increasing quotas or adding more backends

### Issue: Requests Not Load Balancing Between Backends

**Cause:** Load balancer pool not configured correctly or API not using the load balancer

**Solution:**
1. Check APIM > Backends > **OpenAI-LB**
2. Verify the load balancer pool contains both backends:
   - `aoai-primary-backend`
   - `aoai-secondary-backend`
3. Check APIM > APIs > Azure OpenAI Service API > All operations > Policies
4. Verify the policy uses: `<set-backend-service backend-id="OpenAI-LB" />`
5. Check Application Insights to see which backend is handling requests
6. Test by temporarily disabling one backend in the pool to verify failover

### Issue: Can't See Metrics in Application Insights

**Cause:** Diagnostic settings not configured

**Solution:**
1. Go to APIM > **Diagnostic settings**
2. Verify Application Insights is connected
3. Check sampling is set to 100% for testing
4. Wait 5-10 minutes for data to appear

### Issue: HTTP 404 Not Found

**Cause:** Incorrect URL or deployment name

**Solution:**
1. Verify deployment name matches in both Azure OpenAI services
2. Check URL format: `{apim-url}/deployments/{deployment-name}/chat/completions?api-version={version}`
3. Verify API is published in APIM

## Next Steps

Now that you have APIM configured with two Azure OpenAI backends:

1. **Create subscriptions** for different teams or applications
2. **Set up rate limiting** to control costs
3. **Configure monitoring** and alerts
4. **Implement token tracking** for chargeback models
5. **Add more backends** for additional scale
6. **Explore priority routing** for tiered service levels

## Additional Resources

- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Azure API Management Documentation](https://learn.microsoft.com/en-us/azure/api-management/)
- [APIM Policy Reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- [Project GitHub Repository](https://github.com/microsoft/AzureOpenAI-with-APIM)

## Quick Reference: Your Specific Configuration

### Your Deployment Summary

| Component | Value |
|-----------|-------|
| **Primary AI Foundry** | doveia-frc-ai-foundry (France Central) |
| **Secondary AI Foundry** | doveia-sec-ai-foundry-resource (Sweden Central) |
| **Deployment Name** | gpt-4o-mini |
| **Primary URL** | https://doveia-frc-ai-foundry.openai.azure.com/openai |
| **Secondary URL** | https://doveia-sec-ai-foundry-resource.openai.azure.com/openai |
| **APIM Backends** | OpenAI-LB (load balancer pool with both backends) |
| **Load Balancing** | Round-robin distribution with automatic failover |

### How Load Balancing Works

1. **Load Balancer Pool**: All requests go to `OpenAI-LB` backend pool
2. **Round-Robin Distribution**: Requests are distributed evenly between:
   - `doveia-frc-ai-foundry` (France Central)
   - `doveia-sec-ai-foundry-resource` (Sweden Central)
3. **Automatic Failover**: If one backend fails or returns errors:
   - Load balancer automatically routes to the healthy backend
   - No manual intervention needed
4. **Transparent to Client**: Your application doesn't need to handle retries or failover logic

### Test Command (Quick Copy)

**Using Bearer Token (Azure OpenAI SDK compatible):**

```bash
# Replace YOUR_APIM_URL and YOUR_SUBSCRIPTION_KEY
curl "https://YOUR_APIM_URL/deployments/gpt-4o-mini/chat/completions?api-version=2024-02-15-preview" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello!"}
    ],
    "temperature": 0.7,
    "max_tokens": 800
  }'
```

**Or using APIM subscription key header:**

```bash
curl "https://YOUR_APIM_URL/deployments/gpt-4o-mini/chat/completions?api-version=2024-02-15-preview" \
  -H "Content-Type: application/json" \
  -H "Ocp-Apim-Subscription-Key: YOUR_SUBSCRIPTION_KEY" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello!"}
    ],
    "temperature": 0.7,
    "max_tokens": 800
  }'
```

### Monitoring Your Setup

- **APIM Gateway URL**: Azure Portal > APIM > Overview
- **Subscription Keys**: APIM > Subscriptions
- **Backend Status**: APIM > Backends
- **Request Logs**: Application Insights > Transaction search
- **Failures/Retries**: Application Insights > Failures
- **Token Usage**: Log Analytics > Custom queries

### Common Operations

**View Backend Configuration:**
```bash
# In Azure Portal
APIM > Backends > OpenAI-LB (load balancer)
APIM > Backends > aoai-primary-backend (individual backend)
APIM > Backends > aoai-secondary-backend (individual backend)
```

**Check Load Balancer Policy:**
```bash
# In Azure Portal
APIM > APIs > azure-openai-service-api > All operations > Policies (</> icon)
# Look for: <set-backend-service backend-id="OpenAI-LB" />
```

**Monitor Token Usage:**
```kql
// In Log Analytics
customMetrics
| where name in ("Prompt Tokens", "Completion Tokens")
| summarize sum(value) by name, bin(timestamp, 1h)
```

## Conclusion

You now have a production-ready Azure API Management setup with:

- **Two Azure AI Foundry backends** (France Central + Sweden Central) for resiliency
- **Load Balancer Pool** (`OpenAI-LB`) with round-robin distribution
- **Automatic failover** when one backend is unavailable
- **Centralized access control** via subscription keys
- **Usage monitoring and tracking** with Application Insights (optional)
- **Foundation for cost management** and chargeback models
- **gpt-4o-mini load balancing** across two regions

### What You've Achieved

✅ Single APIM endpoint for your applications
✅ Automatic load distribution between France Central and Sweden Central
✅ Automatic failover when one region hits quota limits or is unavailable
✅ Managed identity authentication (no API keys to manage or rotate)
✅ **Azure OpenAI SDK compatibility** - Use Bearer tokens with existing SDK code
✅ Full monitoring and analytics (with Application Insights)
✅ Ready to add rate limiting and cost controls

### Architecture Benefits

- **No manual failover logic** - The load balancer automatically handles distribution and failover
- **Transparent to clients** - Your applications use a single endpoint regardless of which backend responds
- **Easy to scale** - Add more backends to the pool as needed
- **Enterprise-grade** - Secure, reliable, and production-ready

This architecture provides enterprise-grade reliability and management for your Azure OpenAI services.
