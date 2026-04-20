# Azure DevOps Pipelines with RHACS

A demo repo to show how to integrate Red Hat Advanced Cluster Security for Kubernetes CLI (`roxctl`) with Azure DevOps Pipelines.

## Setup

You will need two variables in your ADO pipeline:

| Variable | Description |
|---|---|
| `ROX_API_TOKEN` | API token generated from ACS Central |
| `ROX_CENTRAL_ADDRESS` | URL of your ACS Central instance |

### ROX_API_TOKEN

1. Log into ACS Central as an admin.
2. Go to **Platform Configuration > Integrations > Authentication > StackRox (API Token)**
3. Click **Generate Token**. Give the token a name, choose **Continuous Integration** as the Role, and set an expiration date.
4. Copy the generated value and paste it into the `ROX_API_TOKEN` variable in Azure DevOps.

### ROX_CENTRAL_ADDRESS

This is the URL of your Central instance — usually `central-stackrox.apps.<clusterdomain>:443`. Do **not** include `https://`, but **do** include `:443`.

You should be ready to use ACS to scan your images.

## Sample Pipeline

```yaml
trigger: none

variables:
  IMAGE_REPOSITORY: myorg/myrepo
  IMAGE_TAG: ado
  ROX_VERSION: 4.10.1
  ROXCTL_BIN: $(Pipeline.Workspace)/roxctl-bin

pool:
  vmImage: ubuntu-latest

stages:
  - stage: BuildAndPush
    displayName: Build and Push
    jobs:
      - job: BuildJob
        displayName: Maven Build and Container Push
        steps:
          - checkout: self

          - task: Maven@4
            displayName: Build with Maven
            inputs:
              mavenPomFile: pom.xml
              goals: package
              options: -DskipTests
              publishJUnitResults: false

          - task: Cache@2
            displayName: Cache roxctl
            inputs:
              key: '"roxctl" | "$(ROX_VERSION)"'
              path: $(ROXCTL_BIN)
              cacheHitVar: ROXCTL_CACHE_HIT

          - task: Bash@3
            displayName: Install roxctl
            condition: ne(variables.ROXCTL_CACHE_HIT, 'true')
            inputs:
              targetType: inline
              script: |
                mkdir -p $(ROXCTL_BIN)
                curl -s -L -o $(ROXCTL_BIN)/roxctl \
                  https://mirror.openshift.com/pub/rhacs/assets/$(ROX_VERSION)/bin/Linux/roxctl
                chmod +x $(ROXCTL_BIN)/roxctl

          - task: Docker@2
            displayName: Build and Push to quay.io
            inputs:
              command: buildAndPush
              containerRegistry: quay-service-connection
              repository: $(IMAGE_REPOSITORY)
              dockerfile: Containerfile
              tags: $(IMAGE_TAG)-prescan

          - task: Bash@3
            displayName: roxctl image scan
            env:
              ROX_API_TOKEN: $(ROX_API_TOKEN)
            inputs:
              targetType: inline
              script: |
                $(ROXCTL_BIN)/roxctl image scan \
                  --insecure-skip-tls-verify \
                  -e $(ROX_CENTRAL_ADDRESS) \
                  --image quay.io/$(IMAGE_REPOSITORY):$(IMAGE_TAG)-prescan

          - task: Bash@3
            displayName: roxctl image check
            env:
              ROX_API_TOKEN: $(ROX_API_TOKEN)
            inputs:
              targetType: inline
              script: |
                $(ROXCTL_BIN)/roxctl image check \
                  --insecure-skip-tls-verify \
                  -e $(ROX_CENTRAL_ADDRESS) \
                  --image quay.io/$(IMAGE_REPOSITORY):$(IMAGE_TAG)-prescan

          - task: Docker@2
            displayName: Tag image as ado-dev
            inputs:
              command: push
              containerRegistry: quay-service-connection
              repository: $(IMAGE_REPOSITORY)
              tags: ado-dev
```

The important aspects in the pipeline above are:

| Step | Notes |
|---|---|
| **Cache roxctl** | Not strictly required, but caches the `roxctl` binary so it isn't re-downloaded on every run. |
| **Install roxctl** | Downloads `roxctl` from the Red Hat mirror. Only runs when the cache is cold or the version changes. |
| **roxctl image scan** | Scans the image for vulnerabilities and policy violations. |
| **roxctl image check** | Evaluates scan results against your policies. Fails the build if violations are found. |
