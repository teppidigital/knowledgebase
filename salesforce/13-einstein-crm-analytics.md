# Einstein AI & CRM Analytics

## Category

Salesforce — AI & Data Intelligence

## Context

Salesforce AI has three distinct layers: **Einstein Copilot / Agentforce** (conversational AI assistants and autonomous agents), **CRM Analytics** (formerly Tableau CRM / Einstein Analytics — BI dashboards, dataflows, and SAQL queries), and **Einstein predictive features** (built-in ML models like Lead Scoring, Opportunity Insights, and Next Best Action).

### Einstein AI Product Map

| Product | Type | Purpose |
|---------|------|---------|
| **Einstein Copilot** | Conversational AI | Natural language interface to CRM — built-in Copilot, no code required |
| **Agentforce** | Autonomous AI Agent | Topic-based agents with LLM reasoning + action execution (Apex, Flow, API) |
| **Einstein Prompt Builder** | Prompt engineering | Create/manage reusable LLM prompt templates with Salesforce merge fields |
| **Einstein Prediction Builder** | Custom ML | Point-and-click binary classification model on any CRM object |
| **Next Best Action** | Recommendation | Rules + ML recommendations displayed in Flows or Lightning pages |
| **CRM Analytics** | BI Platform | Dashboards, dataflow-based ETL, SAQL query language, predictive analytics |
| **Einstein Discovery** | Automated ML | AutoML-powered predictive stories + prescriptive recommendations |

### Agentforce Architecture

| Component | Description |
|-----------|-------------|
| **Agent** | Named agent with a system prompt (role, persona, tone instructions) |
| **Topics** | Logical groupings of user intent — each topic has a classification prompt |
| **Actions** | What the agent can do: Apex methods, Flows, Prompt Templates, API calls |
| **Instructions** | Per-topic instructions guiding the LLM towards correct action selection |
| **Channel** | Where the agent is deployed: Copilot, Experience Site, Slack, API |

### CRM Analytics Data Pipeline

| Stage | Tool | Description |
|-------|------|-------------|
| **Extract** | Dataflow / Recipe | Pull from Salesforce objects, connected CSV, or external DB |
| **Transform** | SAQL / Recipe nodes | Filter, augment, join, aggregate datasets |
| **Load** | Dataset | De-normalised columnar store optimised for dashboard queries |
| **Visualise** | Dashboard | Lens + widget-based interactive dashboards with SAQL bindings |

## Pros

- Agentforce actions can call any Apex method marked `@InvocableMethod` — reuses existing business logic without rewriting it as a tool
- Prompt Builder templates support grounding with real-time CRM data via merge fields — reduces hallucination risk
- CRM Analytics Recipes are low-code ETL with visual node-based UI — accessible to non-developers
- Einstein Prediction Builder AutoML takes historical CRM data and produces a production model with no data-science background required
- Next Best Action recommendations can be surfaced natively in Salesforce flows and Lightning components with zero external integration

## Cons

- Agentforce prompt quality is highly dependent on topic classification prompts — poor instructions result in wrong action selection
- LLM actions consume Einstein Credits (AI consumption unit) — cost can escalate at scale without usage monitoring
- CRM Analytics dataset size affects query latency — very large datasets benefit from partitioning by date dimension
- Einstein Prediction Builder models are black-box — model explainability is limited compared to custom ML platforms
- CRM Analytics SAQL has a learning curve and cannot be fully replaced by UI interactions for complex analytics

## Design Diagram

```mermaid
flowchart TD
    subgraph Agentforce Execution
        U[User Query:\n"What loans are overdue for ACME Corp?"]
        COP[Einstein Copilot\nAgent Runtime]
        TOPIC[Topic Classifier\nLLM identifies: Loan Enquiry Topic]
        ACTION[Action: GetOverdueLoans\nApex @InvocableMethod]
        APEX[Apex: SELECT * FROM Loan__c\nWHERE DueDate < TODAY]
        RESPONSE[LLM formats response\nwith retrieved data]
    end

    subgraph CRM Analytics Pipeline
        DF[Dataflow / Recipe\nExtract: Loan__c + Account]
        XFORM[Transform:\nJoin + Compute DaysPastDue\nFilter Status IN Active Overdue]
        DS[Dataset: Loan Performance\nColumnar store]
        DB[Dashboard: Loan Risk Review\nbar chart + KPIs + date filter]
    end

    U --> COP --> TOPIC --> ACTION --> APEX --> RESPONSE
    APEX -->|"Same data"| DF --> XFORM --> DS --> DB
```

## Code Sample

### Apex — `@InvocableMethod` callable by Agentforce action

```apex
public class LoanEnquiryActions {

    public class OverdueLoanInput {
        @InvocableVariable(label='Account Name' required=true)
        public String accountName;

        @InvocableVariable(label='Days Past Due Threshold' required=false)
        public Integer daysPastDue;
    }

    public class OverdueLoanOutput {
        @InvocableVariable(label='Result Summary')
        public String summary;

        @InvocableVariable(label='Overdue Count')
        public Integer overdueCount;
    }

    @InvocableMethod(
        label='Get Overdue Loans for Account'
        description='Returns overdue loan summary for a named account. Used by Einstein Copilot.'
        category='Loan Management'
    )
    public static List<OverdueLoanOutput> getOverdueLoans(List<OverdueLoanInput> inputs) {
        List<OverdueLoanOutput> results = new List<OverdueLoanOutput>();

        for (OverdueLoanInput inp : inputs) {
            Integer threshold = inp.daysPastDue != null ? inp.daysPastDue : 30;
            Date cutoff = Date.today().addDays(-threshold);

            List<Loan__c> overdue = [
                SELECT Id, Name, LoanAmount__c, DueDate__c
                FROM Loan__c
                WHERE Account__r.Name LIKE :('%' + inp.accountName + '%')
                AND DueDate__c < :cutoff
                AND Status__c = 'Active'
                LIMIT 50
            ];

            OverdueLoanOutput out = new OverdueLoanOutput();
            out.overdueCount = overdue.size();
            out.summary = overdue.isEmpty()
                ? 'No overdue loans found for ' + inp.accountName
                : overdue.size() + ' loan(s) past ' + threshold + ' days due for ' + inp.accountName + '.';

            results.add(out);
        }
        return results;
    }
}
```

### JSON — Agentforce topic definition (metadata)

```json
{
  "AgentTopic": {
    "label": "Loan Enquiries",
    "description": "Handles all questions about loan status, overdue amounts, or loan history for a named account or customer.",
    "classificationDescription": "Use this topic when the user asks about loans, repayments, overdue status, or requests loan-related information.",
    "scope": "You are a helpful banking operations assistant. Only discuss loan-related information. Do not discuss account balances, products, or anything outside of the loan portfolio.",
    "actions": [
      {
        "type": "ApexAction",
        "name": "GetOverdueLoans",
        "label": "Get Overdue Loans for Account",
        "description": "Retrieves overdue loans for a specific account name. Use when the user wants to know about overdue loans.",
        "apexClass": "LoanEnquiryActions",
        "apexMethod": "getOverdueLoans"
      }
    ]
  }
}
```

### SAQL — CRM Analytics dataset query for loan risk dashboard

```sql
-- SAQL: Loan Portfolio Risk Analysis
-- Run inside a CRM Analytics lens or dashboard binding

q = load "LoanPerformanceDataset";

-- Filter to active loans with some days past due
q = filter q by Status__c == "Active";

-- Compute risk tier grouping
q = foreach q generate
    AccountName__c as AccountName,
    LoanAmount__c as LoanAmount,
    DaysPastDue__c as DaysPastDue,
    case
        when DaysPastDue__c == 0    then "Current"
        when DaysPastDue__c <= 30   then "1-30 DPD"
        when DaysPastDue__c <= 60   then "31-60 DPD"
        when DaysPastDue__c <= 90   then "61-90 DPD"
        else "90+ DPD"
    end as RiskBucket;

-- Aggregate by risk bucket
q = group q by RiskBucket;
q = foreach q generate
    RiskBucket,
    count() as LoanCount,
    sum(LoanAmount) as TotalExposure;

-- Sort by severity
q = order q by (RiskBucket asc);
q = limit q 10;
```

### TypeScript — Prompt Builder API: invoke a prompt template

```typescript
import axios from 'axios';

interface PromptTemplateRequest {
  templateApiName: string;
  inputBindings: Record<string, string>;
}

async function invokePromptTemplate(
  instanceUrl: string,
  accessToken: string,
  request: PromptTemplateRequest
): Promise<string> {
  const body = {
    promptTemplateName: request.templateApiName,
    inputParams: {
      valueMap: Object.fromEntries(
        Object.entries(request.inputBindings).map(([k, v]) => [
          k,
          { value: { stringValue: v } }
        ])
      )
    }
  };

  const response = await axios.post(
    `${instanceUrl}/services/data/v60.0/einstein/prompt-templates/${request.templateApiName}/generations`,
    body,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
    }
  );

  const generation = response.data?.generations?.[0];
  return generation?.text ?? '';
}

// Usage
const summary = await invokePromptTemplate(
  process.env.SF_INSTANCE_URL!,
  accessToken,
  {
    templateApiName: 'Loan_Risk_Summary_v1',
    inputBindings: {
      AccountName:   'ACME Corp',
      OverdueCount:  '5',
      TotalExposure: '1250000',
    }
  }
);
console.log(summary);
```

## References

- [Agentforce Developer Guide](https://developer.salesforce.com/docs/einstein/genai/guide/get-started.html)
- [Einstein Prompt Builder](https://help.salesforce.com/s/articleView?id=sf.prompt_builder_about.htm)
- [CRM Analytics SAQL Reference](https://developer.salesforce.com/docs/atlas.en-us.bi_dev_guide_saql.meta/bi_dev_guide_saql/bi_saql_intro.htm)
- [Invocable Methods for AI Agents](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableMethod.htm)
- [Einstein Prediction Builder](https://help.salesforce.com/s/articleView?id=sf.bi_edd_prediction_builder.htm)
