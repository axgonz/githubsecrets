# GitHub Secrets and Azure Login

Demonstrate how GitHub Secrets can be used to simplify Azure OIDC login.

This uses a Managed Identity with [Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation) and doesn't require any VM workers in Azure.

> Note: This is [comming soon](https://learn.microsoft.com/en-us/azure/devops/release-notes/roadmap/2022/secret-free-deployments) to DevOps.

## Overview

This sample provides concise steps to:
- Create a user assigned Azure Managed Identity (MSI).
- Federate the MSI with GitHub for use in GitHub Actions.
- Provide the MSI the required permissions to deploy to a target Resource Group.

## Pre-requisites

1. Fork this repository on GitHub.

1. Install Azure CLI [version 2.45.0](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt) or newer and login.

    ``` bash
    # Check version
    az --version 

    # Login
    az login

    # Show active subscription
    az account show
    ```

1. Define these variables (change values as needed).

    ``` bash 
    az_subId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    az_location="australiaeast"
    az_rgName="GitHubFederatedIdentities"
    az_msiName="GitHubActions"

    gh_org="xxxxxx"         # this repo's org or username 
    gh_repo="githubsecrets" # this repo's name
    ```

1. Create target Resource Group for the Managed Identity.

    ``` bash
    # Create Resource Group
    az group create --name $az_rgName --location $az_location --subscription $az_subId
    ```

1. Create a Managed Identity (user assigned) for GitHub to authenticate with Azure. 

    ``` bash
    # Create MSI
    az identity create --name $az_msiName --resource-group $az_rgName --subscription $az_subId

    # Federate with GitHub
    az identity federated-credential create --name "${gh_org}--${gh_repo}" --identity-name $az_msiName --subject "repo:${gh_org}/${gh_repo}:ref:refs/heads/main" --issuer "https://token.actions.githubusercontent.com" --resource-group $az_rgName --subscription $az_subId 

    # Show details (needed to create the AZURE_MSI GitHub secret)
    az identity show --name $az_msiName --resource-group $az_rgName
    ```

1. Assign ARM permissions to the Managed Identity

    ``` bash
    # Assign ARM permissions
    az role assignment create --assignee $az_msiName --role 'Contributor' --scope /subscriptions/$az_subId
    ```

    > Note: The permissions above are an example, use the least privileged approach when assigning permissions.

## Set up

The following uses the provided GitHub workflows to build and deploy the sample.

### Deploy using GitHub

![GitHub workflow](https://github.com/axgonz/githubsecrets/actions/workflows/cicd.yml/badge.svg?branch=main)

1. Use the JSON output from the pre-requisite steps to create a new `AZURE_MSI` GitHub Secret, details in [GitHub Docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets).

    **Name**
    ```
    AZURE_MSI
    ```

    **Value**
    ``` json
    {
        "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx1111",
        "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourcegroups/nxfutil001/providers/Microsoft.ManagedIdentity/userAssignedIdentities/GitHubActions",
        "location": "australiaeast",
        "name": "GitHubActions",
        "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx2222",
        "resourceGroup": "GitHubFederatedIdentities",
        "tags": {},
        "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx3333",
        "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
    }
    ```

1. Run the workflow called `Azure Managed Identity OIDC Login`.

    <img src="./doc/GitHubWorkflow.png" width="300" alt="Running the GitHub Workflow">

### Validate deployment

The workflow in this sample just authenticates and does not deploy anything to the resource group. 

To validate the authentication was successful, check the workflow completed with a successful status:

<img src="./doc/GitHubWorkflowStatus.png" width="500" alt="Running the GitHub Workflow Success">    