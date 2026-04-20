# Azure DevOps Pipenes with RHACS

A demo repo to show how to integrate Red Hat Advanced Cluster Security for Kubernetes CLI (`roxctl`) with Azure DevOps Pipelines.

## Setup

You will need a two variables in your ADO pipeline:
* `ROX_API_TOKEN`: This will be the API token you generate from ACS Central
* `ROX_CENTRAL_ADDRESS`: The URL for your ACS Central instance.

`ROX_API_TOKEN`
* Log into ACS Central as an admin.
* Go to **Platform Configurations -> Integrations -> Authentication -> StackRox (API Token)**
* Click **Generate Token** 