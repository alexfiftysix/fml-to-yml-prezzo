---
layout: center
---

# How?

---

## The assistant is your friend
But not your best friend

![The assistant](/images/assistant.png)

---

## Structure

```text {all|10-12|2-5|6-9|all}
pipelines
├─ jobs
│  ├─ appService.yml
│  ├─ bicep.yml
│  ├─ dbUp.yml
├─ variables
│  ├─ global.yml
│  ├─ dev.yml
│  ├─ test.yml
├─ build.yml
├─ dev.yml
├─ test.yml
```

---

## Code re-use
dev.yml

```yaml {all|2-3,9,13-14|all}
variables:
  - template: variables/global.yml
  - template: variables/dev.yml
  - group: Dev

stages:
  - stage: Infrastructure
    jobs:
      - template: jobs/bicep.yml

  - stage: DeployCode
    jobs:
      - template: jobs/appService.yml
      - template: jobs/dbUp.yml

```

---

## Variables
variables/dev.yml
```yaml {all|2-4|6-11|13-14|all}
variables:
  rootPath: '$(PIPELINE.WORKSPACE)\build-artifact\drop-$(resources.pipeline.build-artifact.runID)'
  databaseAdminLogin: 'remindr-dev'
  environment: Dev

  # bicep
  appServiceName: remindr-app-dev
  appServiceSku: F1
  appServiceHostingPlanName: remindr-app-dev-host
  databaseServerName: remindr-sqlserver-dev
  databaseName: remindr-db-dev

# Secrets are found in the Library tab:
#   databaseAdminPassword

```