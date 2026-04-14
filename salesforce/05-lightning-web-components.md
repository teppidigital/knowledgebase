# Lightning Web Components (LWC)

## Category

Salesforce — Development

## Context

**Lightning Web Components (LWC)** is Salesforce's modern component framework based on Web Standards — custom elements, shadow DOM, and ES modules. Introduced in Spring '19, it replaces the older Aura component model with better performance and closer alignment to the browser's native component model. LWC components run in the browser with Salesforce-provided tooling for security, data access, and platform integration.

### LWC vs Aura vs Visualforce

| Aspect | Visualforce | Aura | LWC |
|--------|-------------|------|-----|
| Technology | Server-rendered HTML | Proprietary JS framework | Web Components (W3C) |
| Performance | Full page reload | Virtual DOM | Shadow DOM, native rendering |
| Data access | Controller (Apex) | `@AuraEnabled` + `$A.enqueueAction` | Wire adapters + Apex |
| Testing | Selenium | Jasmine (manual) | Jest (official) |
| Interop | — | Can contain LWC | Cannot contain Aura |
| Mobile | Limited | Yes | Yes |

### LWC Component Anatomy

```
myComponent/
├── myComponent.html       — Template (HTML + directives)
├── myComponent.js         — Controller (ES module)
├── myComponent.css        — Scoped styles (shadow DOM)
├── myComponent.js-meta.xml — Metadata (targets, API version)
└── __tests__/
    └── myComponent.test.js — Jest unit tests
```

### Data Access Patterns

| Pattern | When to Use | How |
|---------|-------------|-----|
| **Lightning Data Service (LDS)** | Standard CRUD on a single record | `lightning-record-form`, `lightning-record-edit-form` or `getRecord` wire |
| **Wire adapter** | Read data declaratively, auto-refresh | `@wire(getRecord, {...})` |
| **Imperative Apex call** | Complex logic, conditional calls, mutations | `import apexMethod from '@salesforce/apex/...'` + `async/await` |
| **UI API** | Standard object metadata, layouts | `getObjectInfo`, `getPicklistValues` wire adapters |

### Component Communication

| Direction | Mechanism |
|-----------|----------|
| Parent → Child | Public `@api` properties |
| Child → Parent | `CustomEvent` dispatched up, `@api` method called by parent |
| Sibling / cross-tree | Lightning Message Service (LMS) via `MessageChannel` |
| URL-based | NavigationMixin + URL state |

## Pros

- Web Standards compliant — shadow DOM, custom elements; skills transfer beyond Salesforce
- LDS caches record data client-side — reduces Apex server calls for standard objects
- Wire adapters reactively re-run when tracked data changes — no manual refresh logic
- Jest + `@salesforce/sfdx-lwc-jest` enables fast, deterministic unit testing without org
- LWC OSS (Open Source) enables local development and testing without a Salesforce org

## Cons

- Shadow DOM encapsulation breaks CSS cascade — styling child components from parent requires CSS custom properties
- Wire service adapters are read-only — mutations always require imperative Apex calls
- Cannot directly access DOM of child components — must use `@api` methods or events
- `@track` deep reactivity removed in Spring '20 — plain objects now reactive by default but changes must be reassigned
- LWC cannot call Aura-only APIs (e.g., Lightning App Events) — requires LMS instead

## Design Diagram

```mermaid
flowchart LR
    subgraph Browser — Shadow DOM
        P[loanDashboard<br/>parent component]
        C1[loanList<br/>child — @api loans]
        C2[loanDetail<br/>child — @api loanId]
        LMS[Lightning Message Service<br/>MessageChannel]
    end

    subgraph Salesforce Server
        APEX[LoanController.cls<br/>@AuraEnabled methods]
        LDS[Lightning Data Service<br/>UI API]
        SR[Schema / Picklists<br/>UI API]
    end

    P -->|@api loanId| C2
    P -->|@api loans| C1
    C1 -->|CustomEvent - loanselected| P
    P -->|publish| LMS
    C2 -->|subscribe| LMS
    P -->|@wire getLoans| APEX
    C2 -->|imperative: saveRecord| APEX
    C2 -->|@wire getRecord| LDS
    P -->|@wire getPicklistValues| SR
```

## Code Sample

### JavaScript — LWC component with wire + imperative Apex

```javascript
// loanDashboard.js
import { LightningElement, wire, track } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { refreshApex } from '@salesforce/apex';
import getActiveLoans from '@salesforce/apex/LoanController.getActiveLoans';
import updateLoanStatus from '@salesforce/apex/LoanController.updateLoanStatus';

export default class LoanDashboard extends LightningElement {
    @track selectedLoanId;
    @track isLoading = false;

    // Wire: declarative — runs automatically; re-runs when loanStatus changes
    @wire(getActiveLoans, { minAmount: 10000 })
    wiredLoans;  // { data, error } — stored for refreshApex

    get loans() {
        return this.wiredLoans.data ?? [];
    }

    get hasError() {
        return !!this.wiredLoans.error;
    }

    handleLoanSelect(event) {
        this.selectedLoanId = event.detail.loanId;
    }

    async handleApprove() {
        if (!this.selectedLoanId) return;
        this.isLoading = true;
        try {
            // Imperative call — used for mutations
            await updateLoanStatus({
                loanId: this.selectedLoanId,
                newStatus: 'Approved'
            });
            // Refresh the wire cache after mutation
            await refreshApex(this.wiredLoans);
            this.dispatchEvent(new ShowToastEvent({
                title: 'Success',
                message: 'Loan approved successfully',
                variant: 'success'
            }));
        } catch (error) {
            this.dispatchEvent(new ShowToastEvent({
                title: 'Error',
                message: error.body?.message ?? 'Unknown error',
                variant: 'error'
            }));
        } finally {
            this.isLoading = false;
        }
    }
}
```

### HTML — LWC template with directives

```html
<!-- loanDashboard.html -->
<template>
    <lightning-card title="Active Loans" icon-name="standard:loan">
        <div class="slds-p-around_medium">

            <!-- Error state -->
            <template lwc:if={hasError}>
                <p class="slds-text-color_error">Failed to load loans.</p>
            </template>

            <!-- Loading spinner -->
            <template lwc:if={isLoading}>
                <lightning-spinner alternative-text="Loading" size="small"></lightning-spinner>
            </template>

            <!-- Loan list -->
            <template lwc:if={loans.length}>
                <lightning-datatable
                    key-field="Id"
                    data={loans}
                    columns={columns}
                    onrowaction={handleLoanSelect}
                    hide-checkbox-column>
                </lightning-datatable>
            </template>

            <template lwc:else>
                <p>No active loans found.</p>
            </template>

            <!-- Approve button — only if a loan is selected -->
            <template lwc:if={selectedLoanId}>
                <lightning-button
                    label="Approve Loan"
                    variant="brand"
                    onclick={handleApprove}
                    disabled={isLoading}>
                </lightning-button>
            </template>
        </div>
    </lightning-card>
</template>
```

### JavaScript — Child component with `@api` and event dispatch

```javascript
// loanCard.js
import { LightningElement, api } from 'lwc';

export default class LoanCard extends LightningElement {
    @api loanId;
    @api loanName;
    @api loanAmount;
    @api status;

    get statusClass() {
        const map = { Active: 'slds-badge_inverse', Draft: '', Defaulted: 'slds-badge_error' };
        return 'slds-badge ' + (map[this.status] ?? '');
    }

    handleSelect() {
        // Bubble event up to parent
        this.dispatchEvent(new CustomEvent('loanselect', {
            detail: { loanId: this.loanId },
            bubbles: true,      // propagate through shadow DOM
            composed: true      // cross shadow boundary
        }));
    }
}
```

### JavaScript — Lightning Message Service for cross-tree communication

```javascript
// Publisher component
import { LightningElement, wire } from 'lwc';
import { MessageContext, publish } from 'lightning/messageService';
import LOAN_SELECTED_CHANNEL from '@salesforce/messageChannel/LoanSelected__c';

export default class LoanList extends LightningElement {
    @wire(MessageContext) messageContext;

    handleRowSelect(event) {
        publish(this.messageContext, LOAN_SELECTED_CHANNEL, {
            loanId: event.detail.selectedRows[0].Id
        });
    }
}
```

```javascript
// Subscriber component
import { LightningElement, wire } from 'lwc';
import { MessageContext, subscribe, unsubscribe } from 'lightning/messageService';
import LOAN_SELECTED_CHANNEL from '@salesforce/messageChannel/LoanSelected__c';

export default class LoanDetail extends LightningElement {
    @wire(MessageContext) messageContext;
    subscription = null;
    selectedLoanId;

    connectedCallback() {
        this.subscription = subscribe(
            this.messageContext,
            LOAN_SELECTED_CHANNEL,
            (message) => { this.selectedLoanId = message.loanId; }
        );
    }

    disconnectedCallback() {
        unsubscribe(this.subscription);
    }
}
```

### Apex — Controller for LWC (@AuraEnabled)

```apex
public with sharing class LoanController {
    @AuraEnabled(cacheable=true)
    public static List<Loan__c> getActiveLoans(Decimal minAmount) {
        return [
            SELECT Id, Name, LoanAmount__c, InterestRate__c, Status__c,
                   Account__r.Name, OriginationDate__c
            FROM Loan__c
            WHERE Status__c = 'Active'
              AND LoanAmount__c >= :minAmount
            WITH SECURITY_ENFORCED
            ORDER BY LoanAmount__c DESC
            LIMIT 200
        ];
    }

    @AuraEnabled
    public static void updateLoanStatus(Id loanId, String newStatus) {
        Loan__c loan = [SELECT Id, Status__c FROM Loan__c WHERE Id = :loanId WITH SECURITY_ENFORCED];
        loan.Status__c = newStatus;
        update loan;
    }
}
```

## References

- [LWC Developer Guide](https://developer.salesforce.com/docs/component-library/documentation/en/lwc)
- [LWC Recipes (GitHub)](https://github.com/trailheadapps/lwc-recipes)
- [Lightning Data Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.data_ui_api)
- [Lightning Message Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.use_message_channel)
- [LWC Testing with Jest](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.unit_testing_using_jest_introduction)
