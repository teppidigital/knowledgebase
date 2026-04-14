# Salesforce DevOps & CI/CD

## Category

Salesforce — DevOps & Quality

## Context

Salesforce DevOps has matured significantly with **Salesforce DX (SFDX)**, **scratch orgs**, and **unlocked packages**. The modern Salesforce delivery pipeline treats metadata as source code — version-controlled in Git, tested in ephemeral scratch orgs, and deployed through automated pipelines — replacing the legacy sandbox → change set → production approach.

### Delivery Model Comparison

| Model | Environment | Version Control | CI/CD | Recommended For |
|-------|-------------|----------------|-------|----------------|
| **Change Sets** | Sandbox → Production | ❌ None | ❌ Manual | Legacy orgs only |
| **SFDX + Source Format** | Sandbox / Scratch org | ✅ Git | ✅ Manual deploy | Small teams transitioning |
| **Unlocked Packages (2GP)** | Scratch orgs | ✅ Git | ✅ Fully automated | Teams with modular architecture |
| **Managed Packages** | Packaging org | ✅ Git | ✅ | ISV / AppExchange products |
| **DevOps Center** | GUI over SFDX | ✅ Git | ✅ | Admins & mixed teams |

### Scratch Org Lifecycle

```
1. sf org create scratch --definition-file config/project-scratch-def.json
2. sf project deploy start              ← push local source
3. Run Apex tests / LWC Jest tests
4. sf data import tree --plan test-data/plan.json
5. Develop, test locally, commit to Git
6. sf project retrieve start            ← pull org changes back to source
7. Pull Request → CI pipeline validates → merge
8. Scratch org deleted after TTL (1–30 days)
```

### Deployment Pipeline Stages

| Stage | Environment | Gate |
|-------|------------|------|
| **Dev** | Scratch org (per developer) | Unit tests pass locally |
| **PR validate** | CI scratch org | All Apex tests + LWC Jest + PMD lint |
| **Integration** | Shared integration sandbox | Integration tests pass |
| **UAT** | Full sandbox (copy of prod) | Business acceptance |
| **Production** | Production org | ≥75% Apex test coverage, smoke tests |

### Package Types

| Type | Namespace | Subscriber Upgrades | Source-Driven | Best For |
|------|-----------|--------------------|--------------|---------| 
| **Unlocked Package** | Optional | Managed upgrades | ✅ Yes | Internal teams, modular architecture |
| **Managed Package** | Required | Locked (Salesforce controls) | ✅ Yes | ISV / AppExchange distribution |
| **Unlocked (No Namespace)** | None | Manual install | ✅ Yes | Single-org modular builds |

## Pros

- Scratch orgs are disposable, reproducible, and fast — perfect for isolated feature development
- Unlocked packages enforce modular architecture — circular dependencies are blocked at package creation
- CI/CD pipelines catch 75% code coverage failures before they reach production
- SFDX source format is diff-friendly — PRs show real changes, not opaque XML blobs
- Package version pinning enables dependency management across multiple teams

## Cons

- Scratch org setup time (3–5 minutes) slows rapid iteration vs direct sandbox development
- Metadata not yet supported in source format (e.g., some Platform Events configs) requires workarounds
- Unlocked package installation in production can time out on large packages — requires monitoring
- CI runner must be authenticated to a Dev Hub org — secrets management required
- Mixed teams (admin + developer) may resist Git-based workflows — DevOps Center provides the GUI bridge

## Design Diagram

```mermaid
flowchart LR
    subgraph Developer
        DEV[Developer<br/>feature/loan-approval] --> SCR[Scratch Org<br/>local dev + test]
        SCR --> COMMIT[git commit + push]
    end

    subgraph CI Pipeline — GitHub Actions
        COMMIT --> PR[Pull Request]
        PR --> VAL[sf project deploy start<br/>--dry-run validate]
        VAL --> JEST[LWC Jest Tests]
        JEST --> APEX[Apex Tests<br/>sf apex run test]
        APEX --> PMD[PMD Static Analysis]
        PMD --> REVIEW[Code Review]
        REVIEW --> MERGE[Merge to main]
    end

    subgraph Deploy Pipeline
        MERGE --> INT[Deploy to Integration<br/>Sandbox]
        INT --> UAT[Deploy to UAT<br/>Full Sandbox]
        UAT --> PROD[Deploy to Production<br/>≥75% coverage gate]
    end

    subgraph Package Pipeline
        PROD --> PKG[sf package version create<br/>--package LoanManagement --install-key]
        PKG --> INST[Install package version<br/>in subscriber orgs]
    end
```

## Code Sample

### YAML — GitHub Actions CI pipeline for Salesforce

```yaml
# .github/workflows/salesforce-ci.yml
name: Salesforce CI

on:
  pull_request:
    branches: [main, develop]

env:
  SF_CLI_URL: https://developer.salesforce.com/media/salesforce-cli/sf/channels/stable/sf-linux-x64.tar.xz

jobs:
  validate-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Salesforce CLI
        run: |
          wget -q $SF_CLI_URL -O sf-linux-x64.tar.xz
          mkdir -p ~/sf && tar xJf sf-linux-x64.tar.xz -C ~/sf --strip-components 1
          echo "$HOME/sf/bin" >> $GITHUB_PATH

      - name: Authenticate Dev Hub
        run: |
          echo "${{ secrets.DEVHUB_JWT_KEY }}" > server.key
          sf org login jwt \
            --client-id "${{ secrets.DEVHUB_CLIENT_ID }}" \
            --jwt-key-file server.key \
            --username "${{ secrets.DEVHUB_USERNAME }}" \
            --alias DevHub \
            --set-default-dev-hub

      - name: Create scratch org
        run: |
          sf org create scratch \
            --definition-file config/project-scratch-def.json \
            --alias ci-scratch \
            --duration-days 1 \
            --set-default \
            --no-ancestors

      - name: Deploy source
        run: sf project deploy start --target-org ci-scratch

      - name: Import test data
        run: sf data import tree --plan test-data/plan.json --target-org ci-scratch

      - name: Run Apex tests
        run: |
          sf apex run test \
            --target-org ci-scratch \
            --test-level RunLocalTests \
            --code-coverage \
            --result-format human \
            --output-dir test-results/apex \
            --wait 20

      - name: Run LWC Jest tests
        run: npm ci && npm run test:unit -- --coverage

      - name: Run PMD static analysis
        uses: forcedotcom/run-pmd-action@main
        with:
          version: 7.0.0
          rulesets: category/apex/bestpractices.xml,category/apex/security.xml,category/apex/performance.xml

      - name: Delete scratch org
        if: always()
        run: sf org delete scratch --target-org ci-scratch --no-prompt
```

### JSON — Scratch org definition file

```json
{
  "orgName": "Loan Management Dev",
  "edition": "Developer",
  "features": [
    "EnableSetPasswordInApi",
    "API",
    "Communities"
  ],
  "settings": {
    "lightningExperienceSettings": {
      "enableS1DesktopEnabled": true
    },
    "apexSettings": {
      "enableStateAndCountryPicklist": false
    },
    "securitySettings": {
      "sessionSettings": {
        "sessionTimeout": "TwoHours"
      }
    }
  },
  "hasSampleData": false,
  "language": "en_US"
}
```

### JSON — SFDX project configuration (`sfdx-project.json`)

```json
{
  "packageDirectories": [
    {
      "path": "force-app/core",
      "package": "LoanManagement-Core",
      "versionName": "Spring 2026",
      "versionNumber": "2.3.0.NEXT",
      "default": true,
      "dependencies": []
    },
    {
      "path": "force-app/integrations",
      "package": "LoanManagement-Integrations",
      "versionName": "Spring 2026",
      "versionNumber": "1.4.0.NEXT",
      "dependencies": [
        {
          "package": "LoanManagement-Core",
          "versionNumber": "2.3.0.LATEST"
        }
      ]
    }
  ],
  "name": "LoanManagement",
  "namespace": "",
  "sourceApiVersion": "61.0",
  "sfdcLoginUrl": "https://login.salesforce.com"
}
```

### Shell — Package versioning and installation

```bash
# Create a new package version (unlocked package)
sf package version create \
  --package LoanManagement-Core \
  --installation-key "myInstallKey123" \
  --wait 30 \
  --code-coverage

# List package versions
sf package version list --packages LoanManagement-Core

# Install package in target org
sf package install \
  --package "04t..." \
  --installation-key "myInstallKey123" \
  --target-org production \
  --wait 30

# Validate deploy without committing (check only)
sf project deploy validate \
  --manifest manifest/package.xml \
  --target-org StagingOrg \
  --test-level RunLocalTests
```

## References

- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/)
- [Unlocked Packages](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_unlocked_pkg_intro.htm)
- [Salesforce CLI Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/)
- [GitHub Actions for Salesforce](https://github.com/salesforcecli/github-workflows)
- [Salesforce DevOps Center](https://help.salesforce.com/s/articleView?id=sf.devops_center_overview.htm)
