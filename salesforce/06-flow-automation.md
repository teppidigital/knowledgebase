# Flow & Declarative Automation

## Category

Salesforce — Automation

## Context

**Salesforce Flow** is the primary declarative automation tool on the platform. It replaces Workflow Rules and Process Builder (both retired in 2025) and handles the vast majority of automation requirements without code. Flows are metadata, can be version-controlled, and are testable — making them preferable to Apex whenever the logic stays within Flow's capabilities.

### Flow Types

| Flow Type | Trigger | Use Case |
|-----------|---------|---------|
| **Record-Triggered Flow** | Record insert, update, delete | Replace triggers / process builder logic |
| **Scheduled Flow** | Time-based schedule (hourly, daily, etc.) | Batch field updates, reminders, cleanup |
| **Screen Flow** | User interaction (Guided UI) | Multi-step wizards, data collection |
| **Autolaunched Flow** | Apex, REST API, process, another flow | Called programmatically |
| **Platform Event-Triggered** | Platform Event published | Event-driven automation |
| **Orchestration Flow** | Long-running, multi-stage with waits | Approval stages, onboarding sequences |

### Flow vs Apex Decision Matrix

| Criteria | Use Flow | Use Apex |
|----------|----------|----------|
| Simple field updates | ✅ | — |
| Conditional branching / decisions | ✅ | — |
| Record creation / update | ✅ | — |
| HTTP callouts to external systems | ❌ | ✅ |
| Complex business logic, recursion | ❌ | ✅ |
| Bulk processing >10K records reliably | ❌ | ✅ Batch Apex |
| Reusable logic across flows | ✅ Subflow or Action | ✅ |
| Unit testing with code coverage | ❌ (Flow tests separate tool) | ✅ |
| Version control & diffing | ✅ (XML metadata) | ✅ |

### Record-Triggered Flow — Run Order

```
1. Before-save flows (fast, no DML — update same record fields only)
2. System validation rules
3. After-save flows (can create/update other records, send emails, call actions)
4. Apex triggers (after-save flows and Apex triggers share the same execution context)
```

**Before-save flows** are the most efficient — they update the triggering record's fields without an extra DML statement.

## Pros

- No code deployment required — flows deploy as metadata and are active immediately on save
- Flow Builder's visual canvas — business analysts can own and maintain automation
- Subflows enable reuse — build once, reference from multiple parent flows
- `Run As` options: System Context (bypasses sharing), User Context (respects sharing) — per element
- Built-in debug mode with detailed execution trace — no debug logs needed

## Cons

- Flows count against governor limits within the same transaction as Apex triggers — combined limits apply
- Complex flows with many branches are difficult to read and maintain — consider Apex at high complexity
- Flow versioning is coarse — activating a new version deactivates the old one; no A/B deployment
- Error handling in flows is less precise than try/catch in Apex — fault connectors cover element failures
- Scheduled flows have a limit of 250,000 interviews per 24-hour period per org

## Design Diagram

```mermaid
flowchart TD
    subgraph Record-Triggered Flow — Loan Approval
        START([Loan__c Updated\nStatus changed to Active]) --> BSF[Before-Save Flow\nSet ApprovedDate__c = Today]
        BSF --> ASF[After-Save Flow]
        ASF --> D1{LoanAmount > 100K?}
        D1 -->|Yes| TASK[Create Task:\nSenior Review Required]
        D1 -->|No| EMAIL[Send Email Alert\nto Loan Officer]
        TASK --> NOTIFY[Notify via Platform Event]
        EMAIL --> NOTIFY
    end

    subgraph Screen Flow — Loan Application Wizard
        S1[Screen 1: Applicant Details] --> S2[Screen 2: Loan Parameters]
        S2 --> S3[Decision: Credit Score?]
        S3 -->|>= 700| CREATE[Create Loan__c record\nStatus = Draft]
        S3 -->|< 700| DECLINE[Screen: Application Declined]
        CREATE --> SUB[Subflow: CreditCheckSubflow]
        SUB --> S4[Screen 3: Review & Submit]
    end
```

## Code Sample

### XML — Record-Triggered Flow metadata (before-save + after-save)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Flow xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>61.0</apiVersion>
    <label>Loan Approval Processing</label>
    <status>Active</status>
    <processType>AutoLaunchedFlow</processType>
    <triggerType>RecordAfterSave</triggerType>

    <start>
        <locationX>176</locationX>
        <locationY>0</locationY>
        <connector>
            <targetReference>Check_High_Value</targetReference>
        </connector>
        <filterLogic>and</filterLogic>
        <filters>
            <field>Status__c</field>
            <operator>EqualTo</operator>
            <value>
                <stringValue>Active</stringValue>
            </value>
        </filters>
        <object>Loan__c</object>
        <recordTriggerType>Update</recordTriggerType>
        <triggerType>RecordAfterSave</triggerType>
    </start>

    <decisions>
        <name>Check_High_Value</name>
        <label>Is High Value Loan?</label>
        <locationX>176</locationX>
        <locationY>158</locationY>
        <defaultConnectorLabel>Standard Loan</defaultConnectorLabel>
        <rules>
            <name>High_Value_Rule</name>
            <conditionLogic>and</conditionLogic>
            <conditions>
                <leftValueReference>$Record.LoanAmount__c</leftValueReference>
                <operator>GreaterThan</operator>
                <rightValue>
                    <numberValue>100000.0</numberValue>
                </rightValue>
            </conditions>
            <connector>
                <targetReference>Create_Review_Task</targetReference>
            </connector>
            <label>High Value</label>
        </rules>
    </decisions>

    <recordCreates>
        <name>Create_Review_Task</name>
        <label>Create Senior Review Task</label>
        <locationX>44</locationX>
        <locationY>278</locationY>
        <inputAssignments>
            <field>Subject</field>
            <value><stringValue>Senior Review Required</stringValue></value>
        </inputAssignments>
        <inputAssignments>
            <field>WhatId</field>
            <value><elementReference>$Record.Id</elementReference></value>
        </inputAssignments>
        <inputAssignments>
            <field>ActivityDate</field>
            <value><elementReference>$Flow.CurrentDate</elementReference></value>
        </inputAssignments>
        <inputAssignments>
            <field>Priority</field>
            <value><stringValue>High</stringValue></value>
        </inputAssignments>
        <object>Task</object>
    </recordCreates>
</Flow>
```

### Apex — Invoke an Autolaunched Flow from Apex

```apex
// Call a flow programmatically with input variables
public class FlowInvoker {
    public static void runLoanOnboardingFlow(Id loanId, String applicantEmail) {
        Map<String, Object> inputs = new Map<String, Object>{
            'loanId'         => loanId,
            'applicantEmail' => applicantEmail,
            'runDate'        => Date.today()
        };

        Flow.Interview.Loan_Onboarding_Flow interview =
            new Flow.Interview.Loan_Onboarding_Flow(inputs);
        interview.start();

        // Read output variables after flow completes
        String result = (String) interview.getVariableValue('outputStatus');
        System.debug('Flow output status: ' + result);
    }
}
```

### Apex — Test a Flow using `Test.createStub` pattern

```apex
@IsTest
private class LoanApprovalFlowTest {
    @IsTest
    static void testHighValueLoanCreatesTask() {
        // Set up test data
        Account acc = new Account(Name = 'Test Bank');
        insert acc;

        Loan__c loan = new Loan__c(
            Account__c   = acc.Id,
            LoanAmount__c = 250_000,
            Status__c    = 'Draft',
            InterestRate__c = 3.5
        );
        insert loan;

        Test.startTest();
        // Trigger the flow by updating Status to Active
        loan.Status__c = 'Active';
        update loan;
        Test.stopTest();

        // Assert flow created a Task
        List<Task> tasks = [SELECT Subject, Priority FROM Task WHERE WhatId = :loan.Id];
        Assert.isFalse(tasks.isEmpty(), 'Expected a task to be created');
        Assert.areEqual('Senior Review Required', tasks[0].Subject);
        Assert.areEqual('High', tasks[0].Priority);
    }
}
```

### Shell — Deploy and manage flows via SFDX

```bash
# Deploy flow metadata from source
sf project deploy start --source-dir force-app/main/default/flows

# List all active flows in the org
sf data query \
  --query "SELECT MasterLabel, ApiName, ProcessType, Status FROM Flow WHERE Status = 'Active'" \
  --result-format table

# Run flow tests
sf apex run test --class-names LoanApprovalFlowTest --synchronous

# Retrieve flow metadata for version control
sf project retrieve start --metadata "Flow:Loan_Approval_Processing"
```

## References

- [Salesforce Flow Reference](https://help.salesforce.com/s/articleView?id=sf.flow.htm)
- [Flow Best Practices](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_best_practices.htm)
- [Record-Triggered Flow](https://help.salesforce.com/s/articleView?id=sf.flow_ref_elements_triggers.htm)
- [Flow Testing in Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_flow.htm)
- [Migrate from Workflow Rules to Flow](https://help.salesforce.com/s/articleView?id=sf.flow_migrate_from_workflow.htm)
