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

Get the Foundry resource ID, then create a budget scoped to that resource with email alerts
by deploying the included ARM template. This works the same in Bash and PowerShell.

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

### Step 3: Create the budget (ARM template)

Use the included [budget.json](budget.json) ARM template. ARM supplies the api-version itself,
so it works identically in Bash and PowerShell.

1. Edit [budget.json](budget.json): replace `<RESOURCE_ID>` and `<EMAIL>`, and adjust the
   amount and dates.
2. Deploy it to the resource group that contains the Foundry resource:

```
az deployment group create --resource-group "<resource-group>" --template-file budget.json
```

Notes:

- To budget the whole resource group instead of one resource, delete the `filter` block.
- Add a notification block with `"thresholdType": "Forecasted"` for an early warning before
  actual spend reaches the limit.

### Verify the budget in the portal

1. Azure portal, open the resource group that contains the Foundry resource (for example
   `rg-admin-0578`).
2. In the left menu, select **Cost Management**, then **Budgets**.
3. The budget (for example `FoundryBudget`) appears with its amount, Monthly reset, and
   current spend versus budget.
4. Click the budget to see the **ResourceId** filter (scoped to the Foundry resource) and the
   alert thresholds (for example 90% and 100%) with the notification emails.

Alternative path: **Cost Management + Billing** > **Cost Management** > **Budgets**, then set
the scope to the resource group. If it does not appear right away, refresh and confirm the
Budgets blade scope matches the resource group you deployed to.

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
