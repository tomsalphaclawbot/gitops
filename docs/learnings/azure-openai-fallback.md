# Azure OpenAI Regional Fallback (BYOK)

How to set up multi-region failover with your own Azure OpenAI credentials so your assistants survive rate limits and regional outages.

---

## How It Works

When you provide your own Azure OpenAI credentials, Vapi routes LLM requests exclusively through your Azure deployments. Vapi automatically selects the lowest-latency region and falls back to other regions if one becomes unavailable — **but only across regions where you've added credentials.**

A single credential for one region means no failover. To enable regional fallback, create a credential for each Azure region you want available.

---

## Prerequisites

Before configuring Vapi, set up your Azure side:

1. **Deploy the same model(s) in multiple Azure OpenAI resources**, each in a different region. For example:
   - `mycompany-eastus2` in East US 2 with a `gpt-4.1` deployment
   - `mycompany-westus3` in West US 3 with a `gpt-4.1` deployment
   - `mycompany-swedencentral` in Sweden Central with a `gpt-4.1` deployment
2. Each resource needs its own API endpoint, API key, and deployment name(s).

**Important:** Azure deployment names must match the model names Vapi expects. If you want to use `gpt-4.1`, your Azure deployment should be named `gpt-4.1`.

---

## Setup via API

Create one credential per region:

```yaml
# Region 1: East US 2
POST /credential
provider: azure-openai
name: Azure East US 2
region: eastus2
models: [gpt-4.1]
openAIEndpoint: https://mycompany-eastus2.openai.azure.com/
openAIKey: your-eastus2-api-key

# Region 2: West US 3
POST /credential
provider: azure-openai
name: Azure West US 3
region: westus3
models: [gpt-4.1]
openAIEndpoint: https://mycompany-westus3.openai.azure.com/
openAIKey: your-westus3-api-key

# Region 3: Sweden Central
POST /credential
provider: azure-openai
name: Azure Sweden Central
region: swedencentral
models: [gpt-4.1]
openAIEndpoint: https://mycompany-swedencentral.openai.azure.com/
openAIKey: your-swedencentral-api-key
```

If your Azure resources use **Azure API Management**, include `ocpApimSubscriptionKey`:

```yaml
provider: azure-openai
name: Azure East US 2 (APIM)
region: eastus2
models: [gpt-4.1]
openAIEndpoint: https://mycompany-apim.azure-api.net/
openAIKey: your-api-key
ocpApimSubscriptionKey: your-apim-subscription-key
```

---

## Setup via Dashboard

Go to **Dashboard** > **Organization** > **Credentials**:

1. Scroll to the **Azure OpenAI** card
2. Select a **Region** from the dropdown (e.g., `eastus2`)
3. Enter the **OpenAI Endpoint** and **API Key** for that region
4. Select the **Models** deployed in that resource
5. Click **Save**
6. Switch to your next region and repeat

---

## Configuring Your Assistant

Credentials are applied at the **organization level** — you don't link a specific credential to a specific assistant.

**Important:** Set the assistant's model provider to `openai` (not `azure-openai`). Vapi automatically detects your Azure credentials and routes through them:

```yaml
model:
  provider: openai
  model: gpt-4.1
```

### Pinning to a Primary Region (Optional)

For data residency requirements, append the region to the model name:

```yaml
model:
  provider: openai
  model: gpt-4.1:eastus2
```

The pinned region is always tried first. Other credentialed regions still serve as fallbacks. Without pinning, Vapi picks the fastest region automatically.

---

## What Happens at Runtime

On each LLM request:

1. **Identifies available regions** — only regions where the credential's `models` array includes the requested model
2. **Sorts by latency** — continuous latency measurements, fastest-first
3. **Sends to fastest region** — first attempt goes to the lowest-latency region
4. **Falls back on failure** — rate limits, server errors, and timeouts trigger automatic retry on the next-fastest region
5. **Adapts over time** — latency measurements update continuously

### Example

With 3 credentials (eastus2, westus3, swedencentral):
```
Attempt 1: gpt-4.1 via eastus2    → 429 (rate limited)
Attempt 2: gpt-4.1 via westus3    → 200 OK (tokens stream)
           swedencentral is never tried
```

With only 1 credential (eastus2):
```
Attempt 1: gpt-4.1 via eastus2    → 429 (rate limited)
No more regions available          → Call fails with LLM error
```

---

## The `models` Array Matters

Each credential's `models` field must list every model you want routable through that region. If a model isn't listed, that region won't be in the fallback pool for that model.

**Make sure all credentials list the same models** for consistent fallback coverage:

```yaml
# Good — all 3 regions can serve both models
eastus2:        models: [gpt-4.1, gpt-4o]
westus3:        models: [gpt-4.1, gpt-4o]
swedencentral:  models: [gpt-4.1, gpt-4o]

# Problem — swedencentral can't serve gpt-4.1 requests
eastus2:        models: [gpt-4.1, gpt-4o]
westus3:        models: [gpt-4.1, gpt-4o]
swedencentral:  models: [gpt-4o]           # gpt-4.1 missing!
```

---

## Credential Isolation

When you provide Azure OpenAI credentials, **all LLM requests route exclusively through your Azure deployments.** Vapi will never fall back to its own credentials for your calls. This means:

- Your data only flows through endpoints you control
- Your Azure rate limits, content filters, and compliance policies are always respected
- If all your credentialed regions fail, the call receives an LLM error rather than silently routing through another provider

---

## Supported Regions

| Region | Identifier |
|--------|------------|
| Australia East | `australiaeast` |
| Canada East | `canadaeast` |
| Canada Central | `canadacentral` |
| Central US | `centralus` |
| East US | `eastus` |
| East US 2 | `eastus2` |
| France | `france` |
| Germany West Central | `germanywestcentral` |
| India | `india` |
| Japan East | `japaneast` |
| Japan West | `japanwest` |
| North Central US | `northcentralus` |
| Norway | `norway` |
| Poland Central | `polandcentral` |
| South Central US | `southcentralus` |
| Spain Central | `spaincentral` |
| Sweden Central | `swedencentral` |
| Switzerland | `switzerland` |
| UAE North | `uaenorth` |
| UK | `uk` |
| West Europe | `westeurope` |
| West US | `westus` |
| West US 3 | `westus3` |

---

## Important Notes

- **More regions = more resilience.** Two regions is minimum for failover; three or more is recommended for production.
- **Credential changes take effect immediately.** No restart or redeployment needed.
- **Deployment names must match.** The model names in your credential's `models` array must exactly match Vapi's expected model names (e.g., `gpt-4.1`, `gpt-4o`). Vapi validates this when you save the credential.
- **APIM gateway compatibility:** Certain newer API features (such as the OpenAI Responses API) may not be available through Azure API Management gateways. In that case, Vapi falls back to the standard Chat Completions API.
