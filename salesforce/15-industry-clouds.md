# Salesforce Industry Clouds

## Category

Salesforce — Vertical Solutions

## Context

Salesforce **Industry Clouds** are purpose-built extensions of the core platform for regulated and specialised verticals. They deliver pre-built data models, UI components (FlexCards, OmniScript), and business processes so organisations can start with a vertical-aligned foundation rather than building from scratch on Sales or Service Cloud.

### Industry Cloud Landscape

| Industry Cloud | Key Personas | Differentiating Objects |
|---------------|-------------|------------------------|
| **Financial Services Cloud (FSC)** | Relationship managers, advisors, retail & commercial banking | `FinancialAccount__c`, `FinancialAccountTransaction__c`, `FinancialGoal__c`, `ActionPlan`, `RollupSummary`, Person Account model |
| **Health Cloud** | Care coordinators, clinical staff, patients | `CarePlan`, `CareBarrier`, `PatientMedication`, `HealthCondition`, Provider Search |
| **Manufacturing Cloud** | Account managers, demand planners | `SalesAgreement`, `AccountForecast`, `RunRate` |
| **Automotive Cloud** | Dealer staff, OEM marketers | `Vehicle`, `ServiceAppointment`, `DealerPerformance` |
| **Consumer Goods Cloud** | Field reps, key account managers | `Visit`, `AssessmentTask`, `RetailStore` |
| **Public Sector Solutions** | Case workers, constituents | `RecordType`-driven case intake, benefit programmes, inspection management |

### FSC Data Model for Banking

| Object | Purpose |
|--------|---------|
| `PersonAccount` | Individual customer (merges Account + Contact into a single record) |
| `FinancialAccount__c` | Represents a bank account, investment, insurance policy, or loan |
| `FinancialAccountTransactionEvent__c` | Transaction event (CDC-based real-time feed) |
| `FinancialGoal__c` | Savings or investment goals associated with a customer |
| `Household__c` | Groups related Person Accounts into a family household |
| `Rollup Summary` (FinServ Rollups) | Aggregated financial data up from FA → Account → Household |
| `ActionPlan__c` | Templated set of tasks for onboarding or service workflows |
| `RecordAlert__c` | Surfaced notifications (e.g. overdraft threshold crossed, KYC expiry) |

### OmniStudio Components

| Component | Description |
|-----------|-------------|
| **FlexCard** | Lightweight, embeddable summary card for a record — rendered in Lightning pages, communities, or agent desktop |
| **OmniScript** | Guided multi-step wizard — patient intake, loan application, KYC onboarding |
| **DataRaptor** | Extract / transform / load against Salesforce objects without Apex — four types: Extract, Load, Transform, Turbo Extract |
| **Integration Procedure** | Server-side orchestration calling DataRaptors, HTTP actions, and Apex — replaces multiple Apex callouts with a single server-side composition |
| **Decision Matrix** | Tabular rules engine — replaces complex price/eligibility rule Apex |

## Pros

- FSC and Health Cloud reduce data model build time by months — pre-built objects for regulated entities comply with industry terminology (FSA, AML, KYC aligned naming)
- OmniStudio Integration Procedures run entirely server-side — one network round-trip replaces dozens of Apex REST callouts
- Action Plans provide structured, auditable task management for compliance workflows (onboarding checklists, AML review steps)
- Record Alerts surface contextual, role-specific notifications without building a custom notification framework
- Industry Clouds are available as **Managed Packages** — updates are delivered via Salesforce Release Schedule, not manual backport

## Cons

- Industry Clouds add significant managed package metadata — FSC alone introduces 200+ objects and 1000+ fields
- OmniStudio DataRaptors have limited transformational power — complex business logic still requires Apex or Integration Procedures with embedded Apex
- OmniScripts have limited unit testability — end-to-end flow testing requires UI test tools or dedicated test frameworks
- FSC's `PersonAccount` model conflicts with standard `Account + Contact` separation — b2b integrations must be redesigned
- Customising managed package objects requires careful field/relationship naming to avoid namespace collisions on upgrades

## Design Diagram

```mermaid
flowchart TD
    subgraph FSC Banking Data Model
        Personal[PersonAccount\nIndividual Customer\nKYC Status | Risk Rating]
        Household[Household\nFamily Group\nCombined Net Worth\nGoal Progress]
        FA1[FinancialAccount\nType=Loan\nBalance | DueDate]
        FA2[FinancialAccount\nType=Deposit\nBalance | Interest Rate]
        FA3[FinancialAccount\nType=Investment\nPortfolio Value]
        Goal[FinancialGoal\nRetirement\nTarget: £500k]
        Alert[RecordAlert\nKYC Due in 30 days]
        AP[ActionPlan\nOnboarding Checklist\n6 Tasks]
    end

    Household --> Personal
    Personal --> FA1 & FA2 & FA3
    Personal --> Goal
    Personal --> Alert
    Personal --> AP

    subgraph OmniStudio Loan Origination
        OS[OmniScript\nLoan Application Wizard\nStep 1: Personal Details\nStep 2: Documents\nStep 3: Offer Selection]
        DR[DataRaptor Extract\nPre-fill: PersonAccount data]
        IP[Integration Procedure\nCredit Check + Offer Engine\n+ Loan__c Insert]
        OS --> DR
        OS --> IP
    end
```

## Code Sample

### Apex — FSC: create a PersonAccount with FinancialAccount

```apex
public class FscOnboardingService {

    public static Id onboardRetailCustomer(
        String firstName,
        String lastName,
        String email,
        String nationalId
    ) {
        // PersonAccount requires the PersonAccount RecordType
        Id personAccountRtId = Schema.SObjectType.Account
            .getRecordTypeInfosByDeveloperName()
            .get('PersonAccount')
            .getRecordTypeId();

        Account customer = new Account(
            RecordTypeId       = personAccountRtId,
            FirstName          = firstName,
            LastName           = lastName,
            PersonEmail        = email,
            NationalId__c      = nationalId,   // custom KYC field
            KYCStatus__c       = 'Pending'
        );
        insert customer;

        // Link a new Current Account (FinancialAccount) to the customer
        FinServ__FinancialAccount__c fa = new FinServ__FinancialAccount__c(
            Name                              = firstName + ' ' + lastName + ' — Current',
            FinServ__PrimaryOwner__c          = customer.Id,
            FinServ__FinancialAccountType__c  = 'Bank Account',
            FinServ__AccountNumber__c         = generateAccountNumber(),
            FinServ__Status__c                = 'Active',
            FinServ__Balance__c               = 0.00
        );
        insert fa;

        // Assign onboarding Action Plan
        ActionPlan__c ap = new ActionPlan__c(
            Name             = 'KYC Onboarding — ' + lastName,
            ActionPlanTemplate__c = getOnboardingTemplateId(),
            WhatId__c        = customer.Id
        );
        insert ap;

        return customer.Id;
    }

    private static String generateAccountNumber() {
        return 'ACC' + String.valueOf(Math.abs(Crypto.getRandomInteger())).left(8);
    }

    private static Id getOnboardingTemplateId() {
        return [
            SELECT Id FROM ActionPlanTemplate
            WHERE DeveloperName = 'KYC_Retail_Onboarding' LIMIT 1
        ].Id;
    }
}
```

### XML — OmniScript metadata: loan application wizard step

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OmniScript xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>LoanApplication_English_1</fullName>
    <isActive>true</isActive>
    <type>LoanApplication</type>
    <subType>English</subType>
    <elementList>
        <!-- Step 1: Personal Details -->
        <OmniScriptElement>
            <name>PersonalDetailsStep</name>
            <type>Step</type>
            <label>Personal Details</label>
            <children>
                <OmniScriptElement>
                    <name>FirstName</name>
                    <type>Text</type>
                    <label>First Name</label>
                    <required>true</required>
                </OmniScriptElement>
                <OmniScriptElement>
                    <name>LoanAmount</name>
                    <type>Currency</type>
                    <label>Requested Loan Amount</label>
                    <required>true</required>
                    <validationRule>
                        <condition>LoanAmount > 0 AND LoanAmount &lt;= 500000</condition>
                        <errorMessage>Loan amount must be between £1 and £500,000</errorMessage>
                    </validationRule>
                </OmniScriptElement>
            </children>
        </OmniScriptElement>

        <!-- Integration Procedure call for credit check -->
        <OmniScriptElement>
            <name>CreditCheckIP</name>
            <type>IntegrationProcedureAction</type>
            <ipType>LoanOrigination_CreditCheck</ipType>
            <inputMap>{"nationalId": "{%NationalId%}", "loanAmount": "{%LoanAmount%}"}</inputMap>
            <outputMap>{"creditScore": "CreditScore", "eligible": "IsEligible"}</outputMap>
        </OmniScriptElement>
    </elementList>
</OmniScript>
```

### JSON — Integration Procedure definition (summary)

```json
{
  "IntegrationProcedure": {
    "type": "LoanOrigination",
    "subType": "CreditCheck",
    "elements": [
      {
        "type": "DataRaptorExtract",
        "name": "GetCustomerData",
        "dataRaptorType": "Extract",
        "bundleName": "CustomerProfile_Extract",
        "inputMap": { "AccountId": "{%AccountId%}" },
        "outputMap": { "customer": "CustomerData" }
      },
      {
        "type": "HTTPAction",
        "name": "CallCreditBureau",
        "endpoint": "callout:CreditBureau/api/v2/score",
        "method": "POST",
        "inputMap": {
          "nationalId": "{%CustomerData:NationalId__c%}",
          "requestedAmount": "{%loanAmount%}"
        },
        "outputMap": {
          "score": "creditScore",
          "decision": "creditDecision"
        }
      },
      {
        "type": "DataRaptorLoad",
        "name": "SaveCreditResult",
        "bundleName": "CreditResult_Load",
        "inputMap": {
          "AccountId": "{%AccountId%}",
          "CreditScore__c": "{%creditScore%}",
          "CreditDecision__c": "{%creditDecision%}"
        }
      }
    ]
  }
}
```

## References

- [Financial Services Cloud Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.financial_services_cloud_dev_guide.meta/financial_services_cloud_dev_guide/)
- [OmniStudio Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.omnistudio_dev_guide.meta/omnistudio_dev_guide/)
- [Health Cloud Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.health_cloud_dev_guide.meta/health_cloud_dev_guide/)
- [Person Accounts Implementation Guide](https://help.salesforce.com/s/articleView?id=sf.account_person.htm)
- [Salesforce Industry Clouds Overview](https://www.salesforce.com/industries/)
