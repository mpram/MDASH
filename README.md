# MDASH

Operational guides for managing Microsoft AI Foundry models: selecting a model,
limiting tokens and spend, and getting email alerts when a cost threshold is reached.

## Contents

- [1. Select and deploy a model](#1-select-and-deploy-a-model)
- [2. Limit spend and get an email alert](#2-limit-spend-and-get-an-email-alert)
- [3. CLI: get the resource ID and create the budget](#3-cli-get-the-resource-id-and-create-the-budget)
- [Optional: hard stop at the cost limit](#optional-hard-stop-at-the-cost-limit)
- [Quick decision guide](#quick-decision-guide)

There are two separate levers: tokens (throughput, controlled per deployment in
Foundry) and spend (dollars, controlled in Azure Cost Management). Budgets alert you,
they do not hard-stop spend.

## 1. Select and deploy a model

In the [Microsoft Foundry portal](https://ai.azure.com):

1. Open your project, go to **Deployments** (or **Models + endpoints**).
2. **Deploy model** > **Deploy base model** > pick the model (for example GPT-4o) > **Confirm**.
3. On the deploy dialog, set the **Tokens per Minute (TPM)** rate limit. This is your first
   throughput cap. TPM adjusts in increments of 1,000 and also sets a proportional
   requests-per-minute limit. Requests over the limit receive HTTP 429.

Change TPM later from **Deployments** > select the deployment > **Edit**, or under
**Management** > **Model quota**.

## 2. Limit spend and get an email alert

Foundry does not bill by dollars, so dollar limits live in **Cost Management**, scoped to the
Foundry resource or its resource group:

1. Azure portal > **Cost Management + Billing** > **Cost Management** > **Budgets** > **Add**.
2. Set **Scope** to the resource group holding your Foundry resource, or filter to the specific
   AI Services resource under **Cost dimension filters**.
3. Set **Reset period** (Monthly), the **Amount**, and the expiration.
4. On the alerts step, add one or more **% thresholds** (for example 50, 90, 100) and the
   **email recipients** (up to 5). You can also add **Forecasted** alerts.
5. Create. Emails arrive within about an hour of the threshold being crossed.

Add `azure-noreply@microsoft.com` to safe senders so alerts do not go to junk.
Reference: [Tutorial: Create and manage budgets](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets).

> A Cost Management budget cannot target a single model deployment. The finest scope is the
> whole Foundry (Azure AI Services) resource. For per-model dollar control, isolate the model
> in its own resource.

## 3. CLI: get the resource ID and create the budget

Works in Azure Cloud Shell (bash) or the Azure CLI on any machine. Creates a budget scoped to
a single AI Foundry resource with email alerts (the reliable way, since the portal picker only
lists resources that already have cost).

### Step 1: Sign in and select the subscription

```bash
az login
az account set --subscription "<subscription-name-or-id>"
```

### Step 2: Get the AI Foundry resource ID

The Foundry resource is a Cognitive Services account. Replace the resource group and account name:

```
az cognitiveservices account show --resource-group "<resource-group>" --name "<foundry-account-name>" --query id -o tsv
```

If you only know the name, find it across the subscription:

```
az resource list --query "[?type=='Microsoft.CognitiveServices/accounts'].{name:name, id:id, rg:resourceGroup}" -o table
```

### Step 3: Create the budget with email alerts

```bash
# --- Fill these in ---
SUB="<subscription-id>"
RG="<resource-group>"
RESOURCE_ID="<paste-resource-id-from-step-2>"
BUDGET_NAME="FoundryBudget"
AMOUNT=1000
EMAIL="customer@contoso.com"
START="2026-07-01T00:00:00Z"
END="2028-06-30T00:00:00Z"

# --- Build the request body ---
cat > budget.json <<JSON
{
  "properties": {
    "category": "Cost",
    "amount": ${AMOUNT},
    "timeGrain": "Monthly",
    "timePeriod": { "startDate": "${START}", "endDate": "${END}" },
    "filter": {
      "dimensions": {
        "name": "ResourceId",
        "operator": "In",
        "values": [ "${RESOURCE_ID}" ]
      }
    },
    "notifications": {
      "Actual_90_Percent": {
        "enabled": true,
        "operator": "GreaterThanOrEqualTo",
        "threshold": 90,
        "thresholdType": "Actual",
        "contactEmails": [ "${EMAIL}" ]
      },
      "Actual_100_Percent": {
        "enabled": true,
        "operator": "GreaterThanOrEqualTo",
        "threshold": 100,
        "thresholdType": "Actual",
        "contactEmails": [ "${EMAIL}" ]
      }
    }
  }
}
JSON

# --- Create the budget ---
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/${SUB}/resourceGroups/${RG}/providers/Microsoft.Consumption/budgets/${BUDGET_NAME}?api-version=2023-11-01" \
  --headers "Content-Type=application/json" \
  --body @budget.json
```

> Shell note: the block above is **Bash** (variable syntax `SUB=...` and the `cat <<JSON`
> heredoc). In Azure Cloud Shell, click **Switch to Bash** first, or use the PowerShell
> version below. Do not mix them: pasting the Bash block into PowerShell fails with errors
> like "Missing expression after unary operator '--'" or "is not recognized as a name of a
> cmdlet".

### Step 3 (PowerShell alternative)

Same result, native to PowerShell. Note the `$` variables and the quoted `"@budget.json"`.

```powershell
$SUB         = "<subscription-id>"
$RG          = "<resource-group>"
$RESOURCE_ID = "<paste-resource-id-from-step-2>"
$BUDGET_NAME = "FoundryBudget"
$AMOUNT      = 1000
$EMAIL       = "customer@contoso.com"
$START       = "2026-07-01T00:00:00Z"
$END         = "2028-06-30T00:00:00Z"

$body = @"
{
  "properties": {
    "category": "Cost",
    "amount": $AMOUNT,
    "timeGrain": "Monthly",
    "timePeriod": { "startDate": "$START", "endDate": "$END" },
    "filter": { "dimensions": { "name": "ResourceId", "operator": "In", "values": [ "$RESOURCE_ID" ] } },
    "notifications": {
      "Actual_90_Percent":  { "enabled": true, "operator": "GreaterThanOrEqualTo", "threshold": 90,  "thresholdType": "Actual", "contactEmails": [ "$EMAIL" ] },
      "Actual_100_Percent": { "enabled": true, "operator": "GreaterThanOrEqualTo", "threshold": 100, "thresholdType": "Actual", "contactEmails": [ "$EMAIL" ] }
    }
  }
}
"@
$body | Set-Content -Path budget.json -Encoding utf8

az rest --method PUT --uri "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Consumption/budgets/$BUDGET_NAME?api-version=2023-11-01" --headers "Content-Type=application/json" --body "@budget.json"
```

Notes:

- The budget alerts on spend, it does not hard-stop usage.
- Emails go to the addresses in `contactEmails` (up to 5).
- Thresholds accept 0.01 to 1000 (percent of the amount). Add a block with
  `"thresholdType": "Forecasted"` for early warning before actual spend hits the limit.
- To budget the whole resource group instead of one resource, delete the `filter` block.
- Keep the whole `az rest` on a single line and the URL in double quotes. If the
  `?api-version=2023-11-01` is dropped (URL wrapped or unquoted), Azure returns
  `MissingApiVersionParameter`.

## Optional: hard stop at the cost limit

A budget alert does not stop usage by itself. To enforce a cutoff, wire the budget to an
**Action Group** that triggers a Logic App or Automation runbook which, at 100%, disables or
deletes the model deployment.
Pattern: [Manage costs with budgets](https://learn.microsoft.com/azure/cost-management-billing/manage/cost-management-budget-scenario).

## Quick decision guide

| Goal | Control |
|------|---------|
| Cap throughput per minute | Deployment **TPM** |
| Dollar email alert | **Cost Management budget** with email thresholds |
| Hard dollar cutoff | Budget **+ action group + Logic App** to disable the deployment |
