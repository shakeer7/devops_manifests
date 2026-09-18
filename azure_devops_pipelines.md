# Azure DevOps Pipelines for Interviews (4+ Years Experience)

## 1. Basic Syntax

**YAML Pipelines, Stages, Jobs, Steps**

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
- stage: Build
  displayName: 'Build and Test Stage'
  jobs:
  - job: BuildJob
    displayName: 'Compile App'
    steps:
    - script: echo "Building the application..."
      displayName: 'Run build script'
    
    - task: DotNetCoreCLI@2
      inputs:
        command: 'build'
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration)'
```

---

## 2. Intermediate Examples

**Conditions, Variable Groups, and Templates**

```yaml
# Using templates for reusable logic
variables:
- group: production-secrets # Pulled from Azure DevOps Library

stages:
- stage: DeployProd
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - template: templates/deploy-job.yml
    parameters:
      environment: 'Production'
      connectionName: 'Azure-Prod-Service-Connection'
      appSettings: $(db_connection_string) # Secret from variable group
```

*templates/deploy-job.yml*
```yaml
parameters:
  - name: environment
    type: string
  - name: connectionName
    type: string
  - name: appSettings
    type: string

jobs:
- job: Deploy
  steps:
  - task: AzureWebApp@1
    inputs:
      azureSubscription: ${{ parameters.connectionName }}
      appName: 'my-web-app-${{ parameters.environment }}'
      appSettings: '-DbConnection "${{ parameters.appSettings }}"'
```
*Note: Understand the difference between runtime variables `$(var)` and compile-time expressions `${{ var }}`.*

---

## 3. Advanced Examples

**Environments, Approvals, and Docker/Kubernetes CI/CD**

```yaml
trigger:
  - main

variables:
  imageName: 'myregistry.azurecr.io/my-app:$(Build.BuildId)'

stages:
- stage: BuildAndPush
  jobs:
  - job: DockerBuild
    steps:
    - task: Docker@2
      inputs:
        containerRegistry: 'ACR-Service-Connection'
        repository: 'my-app'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(Build.BuildId)
          latest

- stage: DeployToAKS
  dependsOn: BuildAndPush
  jobs:
  - deployment: DeployK8s
    environment: 'production.default' # Ties to ADO Environments (Supports manual approvals)
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
          - task: KubernetesManifest@1
            inputs:
              action: 'deploy'
              connectionType: 'kubernetesServiceConnection'
              kubernetesServiceConnection: 'AKS-Prod-Connection'
              manifests: 'k8s/deployment.yaml'
              containers: '$(imageName)'
```

---

## 4. Interview Coding Exercises

### Problem 1: Passing variables between jobs
**Task:** You have a job that generates a version number. You need to use this version number in a completely different job within the same stage.

**Solution:**
```yaml
jobs:
- job: JobA
  steps:
  - bash: |
      VERSION="1.0.42"
      echo "##v0so[task.setvariable variable=myVersion;isOutput=true]$VERSION"
    name: setVarStep

- job: JobB
  dependsOn: JobA
  variables:
    passedVar: $[ dependencies.JobA.outputs['setVarStep.myVersion'] ]
  steps:
  - script: echo "The version from Job A is $(passedVar)"
```

### Problem 2: Terraform Deployment Pipeline
**Task:** Write a simple pipeline to run `terraform init` and `terraform apply`.

**Solution:**
```yaml
steps:
- task: TerraformInstaller@0
  inputs:
    terraformVersion: 'latest'

- task: TerraformTaskV4@4
  displayName: 'Terraform Init'
  inputs:
    provider: 'azurerm'
    command: 'init'
    backendServiceArm: 'My-Azure-Service-Connection'
    backendAzureRmResourceGroupName: 'tfstate-rg'
    backendAzureRmStorageAccountName: 'tfstatesa'
    backendAzureRmContainerName: 'tfstate'
    backendAzureRmKey: 'prod.terraform.tfstate'

- task: TerraformTaskV4@4
  displayName: 'Terraform Apply'
  inputs:
    provider: 'azurerm'
    command: 'apply'
    environmentServiceNameAzureRM: 'My-Azure-Service-Connection'
    commandOptions: '-auto-approve'
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Conditional Syntax
**Scenario:** This step should only run if the variable `DeployFlag` is true. But it's throwing a syntax error.
```yaml
steps:
- script: echo "Deploying!"
  condition: $(DeployFlag) == true
```
**Answer:** The condition syntax requires expression functions. It should be:
```yaml
  condition: eq(variables['DeployFlag'], 'true')
```

### Broken Configuration 2: Service Connection Authorization
**Scenario:** Your pipeline fails on an Azure CLI task with `The pipeline is not valid. Job JobName: Step AzureCLI input connectedServiceNameARM references service connection XYZ which could not be found or you do not have permission to use it.`
**Answer:** Two possible issues:
1. The Service Connection name is misspelled.
2. The Service Connection has not been granted "Pipeline permissions" in Azure DevOps Project Settings -> Service connections -> Security.

---

## 6. Common Interview Questions

**Q: "What is the difference between a Microsoft-hosted agent and a Self-hosted agent?"**
*Answer:* Microsoft-hosted agents are fresh, isolated VMs spun up for each job in Azure. They are maintained by Microsoft but have limitations on software caching and internal network access. Self-hosted agents are VMs (or containers) you maintain on your own infrastructure (or cloud VPC). They are persistent, allow aggressive caching (faster builds), and can access private internal networks/databases without public endpoints.

**Q: "How do you handle approvals for production deployments in YAML pipelines?"**
*Answer:* YAML pipelines use **Environments**. You define an environment in the pipeline (e.g., `environment: 'Production'`). In the Azure DevOps UI, you navigate to Environments, select 'Production', and configure "Approvals and checks" (e.g., requiring specific users/groups to approve before the deployment job executes).

**Q: "Explain parameters vs. variables in ADO pipelines."**
*Answer:* Parameters are resolved at *compile time* (`${{ parameters.name }}`) and can dictate pipeline structure (like looping over jobs). They are strongly typed (string, boolean, object) and can be prompted to the user at runtime. Variables are resolved at *runtime* (`$(variableName)`) and are strictly strings, useful for passing data between steps or from variable groups.

---

## 7. Cheat Sheet

| Syntax | Explanation |
| :--- | :--- |
| `$(VariableName)` | Runtime macro variable evaluation. |
| `${{ variables.Name }}` | Compile-time template expression evaluation. |
| `$[ variables.Name ]` | Runtime expression evaluation (used for conditions and inter-job variables). |
| `resources:` | Block used to trigger pipeline based on other pipelines, Git repos, or Container Registries. |
| `checkout: self` | Explicitly check out the repo (done implicitly by default, but needed if overriding checkout settings). |
| `strategy: matrix` | Run the same job multiple times with different variables (e.g., testing on Node 14, 16, 18). |
