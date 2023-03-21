
## Build
build.yml - Build & publish

```yaml {all|3-5|6-13|15-20|22-30|32-37|all} {maxHeight:'400px'}
...

- job: Publish
  dependsOn: Test
  steps:
    - task: DotNetCoreCLI@2
      displayName: 'Publish API'
      inputs:
        command: 'publish'
        publishWebProjects: false
        projects: 'api/Remindr/Remindr.API/Remindr.Api.csproj'
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/$(Build.BuildId)'
        modifyOutputPath: false

    - task: CopyFiles@2
      displayName: 'copy bicep templates'
      inputs:
        SourceFolder: 'deploy/bicep'
        Contents: '**'
        TargetFolder: '$(Build.ArtifactStagingDirectory)/bicep'

    - task: DotNetCoreCLI@2
      displayName: 'Publish DbUp'
      inputs: 
        command: 'publish'
        publishWebProjects: false
        projects: 'api/Remindr/Remindr.DbUp/Remindr.DbUp.csproj'
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/dbUp -p:PublishSingleFile=True'
        modifyOutputPath: false
        zipAfterPublish: false

    - task: PublishPipelineArtifact@1
      displayName: 'Publish artifact'
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: 'drop-$(Build.BuildId)'
        publishLocation: 'pipeline'
```

<!--
Could publish a "build" artifact - but it downloads slower
-->
