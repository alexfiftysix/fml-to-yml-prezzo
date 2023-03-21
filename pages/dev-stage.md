---
layout: center
---

# Deploying to Dev

---

## Triggers
dev.yml

```yaml {all|1|3-11|all}
trigger: none

resources:
  pipelines:
    - pipeline: build-artifact
      project: Remindr
      source: Build
      trigger:
        branches:
          include:
            - main

...
```

---

## Agent & Variables
dev.yml

```yaml {all|3-4|6-9|all}
...

pool:
vmImage: 'windows-latest'

variables:
  - template: variables/global.yml
  - template: variables/dev.yml
  - group: Dev

...
```

---

## Variables
variables/dev.yml

```yaml {all|2-4|6-11|13-15|17-18|all}
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

  # appsettings
  AppDetails.EnvironmentName: Dev
  ConnectionStrings.RemindrDb: 'Server=tcp:$(databaseServerName).database.windows.net,1433;Initial Catalog=$(databaseName);Persist Security Info=False;User ID=$(databaseAdminLogin);Password=$(databaseAdminPassword);MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;'

# Secrets are found in the Library tab:
#   databaseAdminPassword
```

---

## Stages
dev.yml

```yaml {all|4-7|9-11|13-17|all}
...

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

---

## Set build name
dev.yml

```yaml {4-7}
...

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

---

## Set build name
jobs/setBuildName.yml

```yaml {all|4|6-13|all}
jobs:
  - job: SetBuildName
    steps:
      - checkout: none

      - task: PowerShell@2
        displayName: 'Set the name of the build'
        inputs:
          targetType: 'inline'
          script: |
            [string] $buildName = "$(resources.pipeline.build-artifact.runName)-$(Build.BuildId)"
            Write-Host "Setting the name of the build to '$buildName'."
            Write-Host "##vso[build.updatebuildnumber]$buildName"
```

<!--
Have to use this script
<br>
There is a 'name' variable which can be set, but it can only be set with compile-time variables. 
-->

---

## Deploy the infrastructure
dev.yml

```yaml {9-11}
...

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

---

## Deploy the infrastructure
jobs/bicep.yml

```yaml {all|2-3|4-5|8|10-26|all} {maxHeight:'400px'}
jobs:
  - deployment: Deploy
    environment: $(environment)
    strategy:
      runOnce:
        deploy:
          steps:
            - checkout: none

            - task: AzureResourceManagerTemplateDeployment@3
              displayName: 'Deploy bicep template'
              inputs:
                connectedServiceName: $(serviceConnectionName)
                deploymentName: $(Build.BuildNumber)
                location: $(deploymentDefaultLocation)
                resourceGroupName: $(resourceGroupName)
                csmFile: '$(rootPath)/bicep/main.bicep'
                overrideParameters: >
                  -appServiceLinuxFxVersion $(runtimeStack)
                  -appServiceName $(appServiceName)
                  -appServiceSku $(appServiceSku)
                  -appServiceHostingPlanName $(appServiceHostingPlanName)
                  -databaseServerName $(databaseServerName)
                  -databaseName $(databaseName)
                  -databaseAdminLogin $(databaseAdminLogin)
                  -databaseAdminPassword $(databaseAdminPassword)
```

<!--
'deployment' instead of 'step' means that we go through the approval process for the Environment & The build is associated with an environment
--> 

---

## Bicep
deployment/bicep/main.bicep
```bicep
param location string = resourceGroup().location

param appServiceName string = 'remindr-app-dev'
param appServiceSku string = 'F1'
param appServiceHostingPlanName string = 'remindr-app-dev-host'
param appServiceLinuxFxVersion string = 'DOTNETCORE|7.0'

param databaseServerName string = 'reminder-sqlserver-dev'
param databaseName string = 'reminder-db-dev'
param databaseAdminLogin string = 'reminder-dev'
@secure()
param databaseAdminPassword string

module appService 'app-service.bicep' = {
  name: 'app-service'
  params: {
    location: location
    hostingPlanName: appServiceHostingPlanName
    name: appServiceName
    sku: appServiceSku
    linuxFxVersion: appServiceLinuxFxVersion
  }
}

module database 'database.bicep' = {
  name: 'database'
  params: {
    location: location
    serverName: databaseServerName
    sqlDBName: databaseName
    administratorLogin: databaseAdminLogin
    administratorLoginPassword: databaseAdminPassword
  }
}
```

---

## Deploy the code
dev.yml

```yaml {13-17}
...

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

---

## Deploy the app service
jobs/appService.yml

```yaml {all|2|8|10-14|16-24|all} {maxHeight:'400px'}
jobs:
  - deployment: Deploy
    environment: $(environment)
    strategy:
      runOnce:
        deploy:
          steps:
            - checkout: none

            - task: FileTransform@1
              inputs:
                folderPath: '$(rootPath)/$(resources.pipeline.build-artifact.runID)/$(resources.pipeline.build-artifact.runID).zip'
                fileType: json
                targetFiles: '**/appsettings.json'

            - task: AzureRmWebAppDeployment@4
              displayName: 'Deploy app service'
              inputs:
                ConnectionType: 'AzureRM'
                azureSubscription: $(subscription)
                appType: 'webAppLinux'
                WebAppName: $(appServiceName)
                packageForLinux: '$(rootPath)/$(resources.pipeline.build-artifact.runID)/$(resources.pipeline.build-artifact.runID).zip'
                RuntimeStack: $(runtimeStack)
```

---

## Migrate that database
jobs/dbUp.yml

```yaml {all|4|6|8-13|15-19|all}
jobs:
  - job: DbUp
    steps:
      - checkout: none

      - download: build-artifact

      - task: FileTransform@2
        displayName: 'DbUp file transform'
        inputs:
          folderPath: '$(rootPath)/dbUp'
          jsonTargetFiles: '**\appsettings.json'
          xmlTransformationRules: '' # Make sure it doesn't try to xml - removes a silly error message

      - task: PowerShell@2
        displayName: 'Run DbUp'
        inputs:
          targetType: 'inline'
          script: '$(rootPath)/dbUp/Remindr.DbUp'
```

---

## Putting it all together
dev.yml

```yaml {all} {maxHeight:'400px'}
trigger: none

resources:
  pipelines:
    - pipeline: build-artifact
      project: Remindr
      source: Build
      trigger:
        branches:
          include:
            - main

pool:
  vmImage: 'windows-latest'

variables:
  - template: variables/global.yml
  - template: variables/dev.yml
  - group: Dev

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

