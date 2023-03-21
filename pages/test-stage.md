---
layout: center
---

# Deploying to Test
Or any environment you feel like

---

## Here's the whole thing
test.yml

```yaml {all|13,14|all} {maxHeight:'400px'}
trigger: none

resources:
  pipelines:
    - pipeline: build-artifact
      source: Build

pool:
  vmImage: 'windows-latest'

variables:
  - template: variables/global.yml
  - template: variables/test.yml
  - group: Test

stages:
  - stage: SetBuildName
    displayName: 'Set build name'
    jobs:
      - template: jobs/setBuildName.yml

  - stage: Infrastructure
    jobs:
      - template: jobs/bicep.yml

  - stage: DeployCode
    displayName: 'Deploy code'
    jobs:
      - template: jobs/appService.yml
      - template: jobs/dbUp.yml

```
