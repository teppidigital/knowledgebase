# Apex Development & Governor Limits

## Category

Salesforce — Development

## Context

**Apex** is Salesforce's strongly-typed, Java-like server-side programming language. It executes in a multi-tenant environment where every transaction is constrained by **governor limits** — hard resource caps enforced per transaction (synchronous) and per 24-hour period (async). Writing correct Apex means designing around these limits from the start, not as an afterthought.

### Governor Limits — Synchronous Context

| Limit | Cap | Common Violation |
|-------|-----|-----------------|
| SOQL queries | 100 | Queries inside loops |
| SOQL rows returned | 50,000 | Unbounded queries |
| DML statements | 150 | DML inside loops |
| DML rows | 10,000 | Large unhandled collections |
| CPU time | 10,000 ms | Nested loops, excessive computation |
| Callouts (HTTP) | 100 | Multiple external calls per transaction |
| Heap size | 6 MB | Storing large lists in memory |
| Future method calls | 50 | Chaining too many async operations |

### Async Apex Types

| Type | When to Use | Governor Limits |
|------|------------|----------------|
| **`@future`** | Simple async callout or DML after a trigger | Sync limits; no chaining |
| **Queueable** | Chaining, passing complex data, monitoring | Higher heap; can chain |
| **Batch Apex** | Large data set processing (>10K records) | Separate per-batch limits |
| **Schedulable** | Cron-style recurring execution | Delegates to Batch or Queueable |
| **Platform Event trigger** | React to platform events asynchronously | Own limits per event |

### Trigger Best Practices

The **one trigger per object** pattern delegates logic to a dedicated handler class, enabling testability, order control, and bypass switches.

```
Trigger → TriggerHandler dispatches to → Service classes
```

## Pros

- Full object-oriented language with interfaces, inheritance, and generics (limited)
- Deep Salesforce API access — SOQL, DML, metadata, ConnectAPI — not available in flows
- Testable with `@isTest` classes achieving required 75% coverage gate
- Async patterns (Batch, Queueable) enable processing tens of millions of records safely
- Strongly typed: compile-time errors catch SOQL and DML mistakes before deployment

## Cons

- Governor limits require discipline — every SOQL/DML must be outside loops (bulkification)
- Apex is not open-source — limited to Salesforce org execution; no local unit testing without mocking
- Debugging is difficult in production — only debug logs with verbose levels; no breakpoints
- Limited language features vs Java/C# — no lambdas (pre-Spring '24), no generics on collections
- Callout limits and no parallel execution make integrations sequential unless carefully designed

## Design Diagram

```mermaid
flowchart TD
    subgraph Trigger Entry Point
        T[AccountTrigger.trigger\nbefore insert / after update]
    end

    subgraph Handler Layer
        TH[AccountTriggerHandler\ndispatch by context]
    end

    subgraph Service Layer
        S1[AccountService\nbusiness logic]
        S2[CreditCheckService\nexternal callout - future/Queueable]
    end

    subgraph Repository Layer
        R1[AccountRepository\nSOQL queries]
        R2[ContactRepository\nbulk DML]
    end

    T --> TH
    TH -->|beforeInsert| S1
    TH -->|afterInsert| S2
    S1 --> R1
    S1 --> R2
    S2 -->|@future / Queueable| EXT[External Credit API\nHTTP Callout]
```

## Code Sample

### Apex — One trigger per object with handler dispatch

```apex
// AccountTrigger.trigger
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    new AccountTriggerHandler().run();
}
```

```apex
// AccountTriggerHandler.cls
public class AccountTriggerHandler extends TriggerHandler {
    private List<Account> newList;
    private Map<Id, Account> oldMap;

    public AccountTriggerHandler() {
        this.newList = (List<Account>) Trigger.new;
        this.oldMap  = (Map<Id, Account>) Trigger.oldMap;
    }

    protected override void afterInsert() {
        AccountService.onAfterInsert(newList);
    }

    protected override void afterUpdate() {
        AccountService.onAfterUpdate(newList, oldMap);
    }
}
```

### Apex — Bulkified service layer (no SOQL/DML in loops)

```apex
public class AccountService {
    // Called from trigger handler with full list — never per-record
    public static void onAfterInsert(List<Account> accounts) {
        // Collect Ids for bulk query
        Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();

        // Single SOQL — never inside a loop
        Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
        for (Contact c : [
            SELECT Id, AccountId, Email
            FROM Contact
            WHERE AccountId IN :accountIds
        ]) {
            if (!contactsByAccount.containsKey(c.AccountId)) {
                contactsByAccount.put(c.AccountId, new List<Contact>());
            }
            contactsByAccount.get(c.AccountId).add(c);
        }

        // Build DML list — never DML inside a loop
        List<Task> tasksToInsert = new List<Task>();
        for (Account acc : accounts) {
            if (acc.AnnualRevenue > 1_000_000) {
                tasksToInsert.add(new Task(
                    Subject     = 'High-value onboarding call',
                    WhatId      = acc.Id,
                    ActivityDate = Date.today().addDays(1),
                    Status      = 'Not Started',
                    Priority    = 'High'
                ));
            }
        }

        if (!tasksToInsert.isEmpty()) {
            insert tasksToInsert;  // Single DML statement
        }
    }
}
```

### Apex — Queueable for chained async processing with callout

```apex
public class CreditCheckQueueable implements Queueable, Database.AllowsCallouts {
    private List<Id> accountIds;

    public CreditCheckQueueable(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        List<Account> accounts = [
            SELECT Id, Name, AnnualRevenue, ExternalAccountId__c
            FROM Account
            WHERE Id IN :accountIds
        ];

        List<Account> toUpdate = new List<Account>();

        for (Account acc : accounts) {
            HttpRequest req = new HttpRequest();
            req.setEndpoint('callout:CreditBureau/scores/' + acc.ExternalAccountId__c);
            req.setMethod('GET');
            req.setTimeout(10_000);

            HttpResponse res = new Http().send(req);
            if (res.getStatusCode() == 200) {
                Map<String, Object> body =
                    (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
                acc.CreditScore__c = (Decimal) body.get('score');
                toUpdate.add(acc);
            }
        }

        if (!toUpdate.isEmpty()) {
            update toUpdate;
        }
    }
}
```

### Apex — Batch Apex for large data volume processing

```apex
public class LoanInterestBatch implements Database.Batchable<SObject>, Database.Stateful {
    private Integer processedCount = 0;

    public Database.QueryLocator start(Database.BatchableContext ctx) {
        return Database.getQueryLocator(
            'SELECT Id, LoanAmount__c, InterestRate__c, AccruedInterest__c ' +
            'FROM Loan__c WHERE Status__c = \'Active\''
        );
    }

    public void execute(Database.BatchableContext ctx, List<Loan__c> loans) {
        for (Loan__c loan : loans) {
            loan.AccruedInterest__c =
                (loan.AccruedInterest__c ?? 0) +
                (loan.LoanAmount__c * (loan.InterestRate__c / 100) / 365);
        }
        update loans;
        processedCount += loans.size();
    }

    public void finish(Database.BatchableContext ctx) {
        // Notify admin on completion
        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setToAddresses(new List<String>{ 'admin@example.com' });
        mail.setSubject('Interest Batch Complete');
        mail.setPlainTextBody('Processed ' + processedCount + ' loans.');
        Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
    }
}

// Execute with batch size of 200 (max 2000)
Database.executeBatch(new LoanInterestBatch(), 200);
```

### Apex — TriggerHandler base class with bypass switch

```apex
public virtual class TriggerHandler {
    private static Set<String> bypassedHandlers = new Set<String>();

    public void run() {
        String handlerName = getHandlerName();
        if (bypassedHandlers.contains(handlerName)) return;
        if (!isTriggerContext()) return;

        if (Trigger.isBefore) {
            if (Trigger.isInsert) beforeInsert();
            if (Trigger.isUpdate) beforeUpdate();
            if (Trigger.isDelete) beforeDelete();
        } else {
            if (Trigger.isInsert) afterInsert();
            if (Trigger.isUpdate) afterUpdate();
            if (Trigger.isDelete) afterDelete();
            if (Trigger.isUndelete) afterUndelete();
        }
    }

    // Override in subclass
    protected virtual void beforeInsert() {}
    protected virtual void beforeUpdate() {}
    protected virtual void beforeDelete() {}
    protected virtual void afterInsert() {}
    protected virtual void afterUpdate() {}
    protected virtual void afterDelete() {}
    protected virtual void afterUndelete() {}

    // Bypass for data migrations / tests
    public static void bypass(String handlerName) { bypassedHandlers.add(handlerName); }
    public static void clearBypass(String handlerName) { bypassedHandlers.remove(handlerName); }
    public static void clearAllBypasses() { bypassedHandlers.clear(); }

    private String getHandlerName() { return String.valueOf(this).split(':')[0]; }
    private Boolean isTriggerContext() { return Trigger.isExecuting; }
}
```

## References

- [Apex Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [Apex Design Patterns — Trigger Handler](https://github.com/kevinohara80/sfdc-trigger-framework)
- [Queueable Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_queueing_jobs.htm)
- [Batch Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm)
