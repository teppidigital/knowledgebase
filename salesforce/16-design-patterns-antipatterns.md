# Design Patterns & Anti-Patterns

## Category

Salesforce — Architecture

## Context

Salesforce's declarative-first architecture and governor limit model create a distinct set of proven patterns and costly pitfalls. This guide catalogues the most impactful design patterns and anti-patterns observed across enterprise Salesforce implementations — covering Apex, data modelling, automation, integration, and DevOps.

---

## Design Patterns

### 1. One Trigger Per Object (Trigger Framework)

**Pattern:** A single Apex trigger per SObject that delegates all logic to a handler class. The trigger itself contains no business logic.

```apex
// ✅ Pattern: single trigger, no logic, delegates to handler
trigger LoanTrigger on Loan__c (before insert, before update, after insert, after update) {
    LoanTriggerHandler.handle(Trigger.operationType, Trigger.new, Trigger.oldMap);
}

// Handler class — testable, extendable
public class LoanTriggerHandler {
    public static void handle(TriggerOperation op, List<Loan__c> newList, Map<Id, Loan__c> oldMap) {
        switch on op {
            when BEFORE_INSERT  { LoanService.setDefaults(newList); }
            when AFTER_INSERT   { LoanService.createOnboardingTasks(newList); }
            when BEFORE_UPDATE  { LoanService.validateStatusTransition(newList, oldMap); }
            when AFTER_UPDATE   { LoanService.publishStatusChangeEvents(newList, oldMap); }
        }
    }
}
```

**Why:** Multiple triggers on the same object fire in unpredictable order. A single handler gives deterministic, testable execution.

---

### 2. Bulkification — Always Process Collections

**Pattern:** Every Apex method, trigger, and service operates on `List<SObject>` — never a single record. SOQL and DML must always be outside loops.

```apex
// ✅ Pattern: bulk service — SOQL and DML outside the loop
public class LoanService {
    public static void setInterestDefaults(List<Loan__c> loans) {
        List<Loan__c> toUpdate = new List<Loan__c>();
        for (Loan__c loan : loans) {
            if (loan.InterestRate__c == null) {
                loan.InterestRate__c = 3.5;
                toUpdate.add(loan);
            }
        }
        if (!toUpdate.isEmpty()) update toUpdate;  // single DML
    }
}
```

---

### 3. Service Layer Separation

**Pattern:** Business logic lives in a stateless `*Service` class. Triggers and LWC controllers are thin — they call the service. Never put logic directly in triggers or `@AuraEnabled` methods.

```
Trigger → TriggerHandler → LoanService → LoanRepository
LWC → LoanController (@AuraEnabled) → LoanService → LoanRepository
Flow → InvocableMethod → LoanService
```

**Why:** One business rule, one implementation, callable from any entry point (trigger, flow, API, LWC, batch). Simplifies testing.

---

### 4. External ID for Upsert and Integration

**Pattern:** Every custom object that is managed by an external system has an `ExternalId__c` field marked as `Unique` and `External ID`. Use `Database.upsert(records, ExternalId__c, false)` in integrations.

```apex
// ✅ Pattern: upsert by external ID — idempotent, no pre-fetch needed
List<Loan__c> loans = mapFromExternalSystem(payload);
Database.UpsertResult[] results = Database.upsert(loans, Loan__c.ExternalId__c, false);
```

**Why:** Avoids querying for existing records before deciding insert vs update. Makes integrations idempotent — re-running the same payload is safe.

---

### 5. Queueable for Async Work with Chaining

**Pattern:** Use `Queueable` (not `@future`) for asynchronous processing that needs: callout + DML in the same transaction, access to complex types, or job chaining.

```apex
public class CreditCheckQueueable implements Queueable, Database.AllowsCallouts {
    private List<Id> loanIds;

    public CreditCheckQueueable(List<Id> loanIds) {
        this.loanIds = loanIds;
    }

    public void execute(QueueableContext ctx) {
        List<Loan__c> loans = [SELECT Id, ExternalId__c FROM Loan__c WHERE Id IN :loanIds];
        for (Loan__c loan : loans) {
            Integer score = CreditBureauService.getScore(loan.ExternalId__c);
            loan.CreditScore__c = score;
        }
        update loans;
        // Chain next job if needed
        if (moreWorkPending()) System.enqueueJob(new CreditCheckQueueable(nextBatch()));
    }
}
```

---

### 6. Named Credentials for All External Callouts

**Pattern:** Never hardcode endpoint URLs or credentials in Apex. All outbound callouts use Named Credentials — authentication (OAuth, JWT, Basic) is managed by the platform.

```apex
// ✅ Pattern: callout via Named Credential
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:CreditBureau/api/v2/score/' + loanId);
req.setMethod('GET');

// Anti-pattern: hardcoded URL + stored credentials
// req.setEndpoint('https://api.creditbureau.com/score/' + loanId);  ❌
// req.setHeader('X-API-Key', 'hardcoded-key');  ❌
```

**Why:** Named Credentials handle token refresh automatically. Credential rotation is done in Setup — zero code change.

---

### 7. Platform Events for Decoupled Integration

**Pattern:** Use Platform Events to decouple Salesforce processes from external systems and from each other. The publisher does not know about subscribers.

```apex
// ✅ Pattern: publish event — publish & forget
EventBus.publish(new LoanStatusChanged__e(
    LoanId__c     = loan.Id,
    NewStatus__c  = loan.Status__c,
    OccurredAt__c = DateTime.now()
));

// Subscriber (trigger on Platform Event) — completely decoupled
trigger LoanStatusChangedTrigger on LoanStatusChanged__e (after insert) {
    for (LoanStatusChanged__e evt : Trigger.new) {
        NotificationService.notifyBorrower(evt.LoanId__c, evt.NewStatus__c);
    }
}
```

---

### 8. Permission Sets Over Profiles

**Pattern:** Grant access via **Permission Sets** (and Permission Set Groups) — not profiles. Profiles are the minimum baseline; all additive access is via Permission Sets.

| Concern | Profile | Permission Set |
|---------|---------|---------------|
| System permissions | ✅ Baseline defaults | Add-on grants |
| Object / Field access | ✅ Baseline defaults | Add-on grants |
| User assignment | One per user | Many per user |
| Packaging | Hard to deploy atomically | Deployable with metadata |
| Licensing | Tied to licence type | Additive |

---

### 9. Scratch Org–Driven Development

**Pattern:** All features are developed in a scratch org created from source. No direct development in sandboxes. All changes live in source control.

```shell
# Pattern: create, develop, test, delete — all from source
sf org create scratch --definition-file config/project-scratch-def.json --alias feature-loan-v2
sf project deploy start --target-org feature-loan-v2
sf apex run tests --target-org feature-loan-v2 --test-level RunLocalTests
sf org delete scratch --target-org feature-loan-v2 --no-prompt
```

---

### 10. Test Data Factory Pattern

**Pattern:** Never insert test data inline in test methods. Use a static `TestDataFactory` class that builds SObjects with all required fields.

```apex
@IsTest
public class TestDataFactory {
    public static Account createAccount(Map<String, Object> overrides) {
        Account acc = new Account(
            Name           = 'Test Account',
            BillingCountry = 'GB',
            Type           = 'Customer'
        );
        for (String field : overrides.keySet()) {
            acc.put(field, overrides.get(field));
        }
        insert acc;
        return acc;
    }

    public static Loan__c createLoan(Id accountId, Map<String, Object> overrides) {
        Loan__c loan = new Loan__c(
            Account__c    = accountId,
            LoanAmount__c = 50000,
            Status__c     = 'Draft',
            InterestRate__c = 3.5
        );
        for (String field : overrides.keySet()) {
            loan.put(field, overrides.get(field));
        }
        insert loan;
        return loan;
    }
}
```

---

## Anti-Patterns

### A1. SOQL or DML Inside a Loop

**Problem:** The most common cause of governor limit exceptions. Each loop iteration consumes a SOQL query or DML statement — with 200 records in a trigger, this immediately hits the 100-query or 150-DML limits.

```apex
// ❌ Anti-pattern: SOQL in a loop
for (Loan__c loan : Trigger.new) {
    Account acc = [SELECT Id, Name FROM Account WHERE Id = :loan.Account__c]; // 1 SOQL per record
    // ...
}

// ✅ Pattern: bulk query before the loop
Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :Trigger.newMap.values()*.Account__c]
);
for (Loan__c loan : Trigger.new) {
    Account acc = accountMap.get(loan.Account__c);
}
```

---

### A2. Logic Directly in Triggers

**Problem:** Business logic in trigger bodies — impossible to test in isolation, no reuse from other entry points, no deterministic ordering across multiple triggers.

```apex
// ❌ Anti-pattern: business logic in trigger body
trigger LoanTrigger on Loan__c (after insert) {
    for (Loan__c loan : Trigger.new) {
        if (loan.Amount__c > 100000) {
            Task t = new Task(Subject = 'Senior Review', WhatId = loan.Id);
            insert t;  // DML inside trigger body — not bulkified
        }
    }
}
```

**Fix:** Use the one-trigger-per-object framework (Pattern 1) and a service layer (Pattern 3).

---

### A3. Storing Credentials in Custom Settings or Custom Metadata

**Problem:** API keys, passwords, or tokens stored in Custom Settings or Custom Metadata are visible to anyone with `Modify All Data` or Setup access, and appear in metadata backups.

**Fix:** Use **Named Credentials** for endpoint + auth. Use **Salesforce Secrets Manager** (or a connected external vault) for sensitive key material.

---

### A4. Hardcoded IDs

**Problem:** Record IDs (RecordType IDs, Profile IDs, Queue IDs) vary between orgs — code using hardcoded IDs fails on deployment to any other org.

```apex
// ❌ Anti-pattern: hardcoded ID
if (loan.RecordTypeId == '0127X000000XXXXX') { ... }

// ✅ Pattern: look up by DeveloperName at runtime
Id personalLoanRtId = Schema.SObjectType.Loan__c
    .getRecordTypeInfosByDeveloperName()
    .get('PersonalLoan')
    .getRecordTypeId();
```

---

### A5. Using `@future` When Queueable Is Needed

**Problem:** `@future` methods cannot be chained, cannot accept SObjects as parameters (only primitives and collections of primitives), and cannot perform HTTP callouts + DML in the same execution if not properly marked.

```apex
// ❌ Anti-pattern: @future with SObject parameter workaround
@future
public static void processSingle(String loanId) { ... }  // loses bulk context

// ✅ Pattern: Queueable — accepts any type, chainable, allows callouts
public class LoanProcessingJob implements Queueable, Database.AllowsCallouts {
    private List<Id> loanIds;
    // ...
}
```

---

### A6. SeeAllData=true in Tests

**Problem:** Tests using `@IsTest(SeeAllData=true)` rely on whatever data exists in the org at the time. Tests pass in one org and fail in another, break after data cleanup, and cannot be run in scratch orgs.

```apex
// ❌ Anti-pattern
@IsTest(SeeAllData=true)
private class LoanTest {
    @IsTest static void test() {
        Loan__c loan = [SELECT Id FROM Loan__c LIMIT 1];  // depends on org state
    }
}

// ✅ Pattern: SeeAllData=false (default) + TestDataFactory
@IsTest
private class LoanTest {
    @TestSetup static void setup() { TestDataFactory.createLoan(...); }
}
```

---

### A7. Over-relying on Workflow Rules / Process Builder

**Problem:** Salesforce has retired Process Builder and is sunset-ing Workflow Rules. Orgs with hundreds of legacy Workflow Rules and Process Builders have unmaintainable automation layers that interact unpredictably.

**Fix:** Migrate to **Record-Triggered Flows** (declarative) or **Apex** (code). Use the `before save` flow context for field updates — it executes before triggers and avoids extra DML.

---

### A8. Deploying Without Apex Test Execution

**Problem:** Deploying metadata without running tests bypasses the 75% code coverage gate. This works in sandboxes via `--test-level NoTestRun` but will fail in production and masks existing test failures.

```shell
# ❌ Anti-pattern: skipping tests in CI
sf project deploy start --test-level NoTestRun --target-org sandbox

# ✅ Pattern: always run local tests in CI; RunAllTestsInOrg in production pipeline
sf project deploy start --test-level RunLocalTests --target-org sandbox
sf project deploy start --test-level RunAllTestsInOrg --target-org prod
```

---

### A9. Making Every Field Required in the Schema

**Problem:** Over-constrained data models with excessive `Required` field settings cause integration failures — inbound data from external systems rarely provides every field, resulting in validation rule exceptions that block the entire upsert.

**Fix:** Required constraints belong in **validation rules** with conditional logic (`ISNEW()`, `ISPICKVAL()`, `$Profile.Name`). This allows phased data entry, system integrations, and role-based requirements.

---

### A10. Ignoring FLS in Apex

**Problem:** Apex runs in system context by default — it bypasses Field-Level Security. A resolver that reads or writes fields the current user cannot access will expose or corrupt data silently.

```apex
// ❌ Anti-pattern: no FLS enforcement
List<Loan__c> loans = [SELECT Id, InterestRate__c, InternalCreditScore__c FROM Loan__c];
// InternalCreditScore__c may be hidden from this user — retrieved anyway

// ✅ Pattern: stripInaccessible before returning to presentation layer
SObjectAccessDecision dec = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, InterestRate__c, InternalCreditScore__c FROM Loan__c]
);
List<Loan__c> safeLoans = (List<Loan__c>) dec.getRecords();
```

---

## Quick Reference Card

| # | Pattern / Anti-pattern | Verdict | Key Rule |
|---|----------------------|---------|---------|
| P1 | One trigger per object | ✅ Pattern | Delegate all logic to a handler class |
| P2 | Bulkification | ✅ Pattern | SOQL and DML always outside loops |
| P3 | Service layer separation | ✅ Pattern | Logic in services — not triggers or controllers |
| P4 | External ID for integration | ✅ Pattern | Idempotent upsert by external key |
| P5 | Queueable for async | ✅ Pattern | Chainable, supports complex types |
| P6 | Named Credentials | ✅ Pattern | Never hardcode endpoints or secrets |
| P7 | Platform Events for decoupling | ✅ Pattern | Publisher knows nothing about subscribers |
| P8 | Permission Sets over Profiles | ✅ Pattern | Profiles = baseline; Perm Sets = additive |
| P9 | Scratch org–driven development | ✅ Pattern | All dev in scratch orgs from source |
| P10 | Test data factory | ✅ Pattern | No inline data; no SeeAllData=true |
| A1 | SOQL/DML in a loop | ❌ Anti-pattern | Governor limit exception waiting to happen |
| A2 | Logic in trigger body | ❌ Anti-pattern | Untestable, unordered, non-reusable |
| A3 | Credentials in Custom Metadata | ❌ Anti-pattern | Use Named Credentials / external vault |
| A4 | Hardcoded record IDs | ❌ Anti-pattern | Look up by DeveloperName at runtime |
| A5 | `@future` for complex async | ❌ Anti-pattern | Use Queueable instead |
| A6 | `SeeAllData=true` in tests | ❌ Anti-pattern | Tests must own their data |
| A7 | Process Builder / Workflow Rules | ❌ Anti-pattern | Migrate to Record-Triggered Flows |
| A8 | Deploy without tests | ❌ Anti-pattern | Always run tests in CI |
| A9 | Every field required | ❌ Anti-pattern | Use conditional validation rules |
| A10 | Ignoring FLS in Apex | ❌ Anti-pattern | Use `Security.stripInaccessible` |

## References

- [Apex Design Patterns — Salesforce Architects](https://architect.salesforce.com/design/decision-guides/trigger-automation)
- [Salesforce Well-Architected Framework](https://architect.salesforce.com/well-architected/overview)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [Named Credentials](https://help.salesforce.com/s/articleView?id=sf.named_credentials_about.htm)
- [Security.stripInaccessible](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_with_security_stripInaccessible.htm)
