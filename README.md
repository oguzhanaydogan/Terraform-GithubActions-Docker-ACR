# Tetris Game Deployment Pipeline
This project contains the necessary code to deploy a Tetris game application using Terraform and GitHub Actions. The infrastructure is deployed to Azure using the azurerm provider and consists of an Azure Container Registry, an Azure Web App, and an Azure Resource Group. The Docker image of the game is built and pushed to the Azure Container Registry, and then deployed to the Azure Web App.

## Pipeline Overview
The pipeline consists of three stages: Terraform plan, Terraform apply, and Build/Push Docker Image. Here is an overview of each stage:

1. Terraform Plan - The first stage of the pipeline uses the terraform plan command to create an execution plan. This plan shows what changes Terraform will make to the infrastructure when it is applied. The plan is then saved to a file named tfplan. If the plan is successful, the pipeline continues to the next stage.

1. Terraform Apply - The second stage of the pipeline uses the terraform apply command to apply the changes shown in the execution plan created in the previous stage. If the apply is successful, the pipeline continues to the next stage.

1. Build/Push Docker Image - The final stage of the pipeline builds and pushes the Docker image of the Tetris game to the Azure Container Registry, and then deploys the image to the Azure Web App.

## Setting Credentials
Terraform needs Azure Credentials to create the infrastructure. We need to provide these values in environment for Terraform to look up.
- ARM_SUBSCRIPTION_ID
- ARM_TENANT_ID
- ARM_CLIENT_ID
- ARM_CLIENT_SECRET
- AZURE_SERVICE_PRINCIPAL 

To get these credentials we use this command;
```
az ad sp create-for-rbac --sdk-auth --role="Contributor" --scopes="/subscriptions/<subscription_id>"
```

Terraform also needs GitHub Token to create the Variables in GitHub repository. We provide the token securely by defining it in the GitHub Actions Secrets as `GH_TOKEN`. We assign this value in the pipeline environment section to `GITHUB_TOKEN` with:
```
GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
```

## Terraform Configuration

The Terraform configuration file main.tf is located in the infrastructure directory. The configuration file uses the azurerm and github providers to create the necessary infrastructure. The following resources are created:

- An Azure Resource Group.
- An Azure Container Registry.
- An Azure App Service Plan.
- An Azure Web App.

The Azure Container Registry is used to store the Docker image of the Tetris game, which is built and pushed to the registry during the pipeline. The Azure Web App is used to host the game.

Since GitHub Actions Pipeline uses an ephemeral agent we need to define a backend to keep our `terraform.tfstate`. We configure our backend to Azure Blob Container with following code 
```
  backend "azurerm" {
    resource_group_name  = "<resource-group-name>"
    storage_account_name = "<storage-account-name>"
    container_name       = "<container-name>"
    key                  = "terraform.tfstate"
  }
```

In order to access to our `Azure Container Registry` we need to set 
```
admin_enabled       = true
```
in the `azurerm_container_registry` block.

To use later in the pipeline we define a `github_actions_environment_secret` 
- ACR_PASSWORD

and multiple `github_actions_environment_variable`
- ACR_USERNAME
- RESOURCEGROUP
- WEBAPP

## GitHub Actions Configuration
The pipeline is configured using GitHub Actions. The configuration file main.yml is located in the .github/workflows directory. The pipeline is triggered by pushes to the main branch and pull requests against the main branch.

Since we have our Terraform configuration files in a dedicated folder, we need to define this path in the job environment for the steps which need to access to this folder to run 
```
env:
     working-directory: infrastructure/
``` 

The Sleep Step is necessary to allow some time for "Web App" to get ready.
```
      name: Sleep for 5 minutes
      run: sleep 5m
      shell: bash
```

Web App needs access to the ACR to pull the image. For that purpose we need to inject the necessary credentials into Web App configuration with 
```
name: 'Set private registry authentication settings'
run: az webapp config container set --name ${{ vars.WEBAPP }} --resource-group ${{ vars.RESOURCEGROUP }} \                 
      --docker-registry-server-user ${{ secrets.ACR_USERNAME }} \
      --docker-registry-server-password ${{ secrets.ACR_PASSWORD }}
```
