# Azure Nextflow

This repository contains sample only code to demonstrate how secrets can be marshaled from Key Vault into a Container Instance running Nextflow for the purpose of dispatching Nextflow pipelines on Azure Batch.

## Overview

Deploying the Azure infrastructure supporting this sample is left to you.

It is assumed that this infrastructure would be deployed through a DevOps pipeline of GitHub workflow. 

This sample will be eventually expanded to include Bicep templates for capturing the state of required Azure resources and pipeline/workflow files for the deployment.

``` bash
loc="australiaeast"
sub="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
az deployment sub create --name "batch-$loc" --location $loc --subscription $sub --template-file ./azure/templates/main.bicep
```

## Set up

As per above, it is assumed (for now) that the following Azure resources are already deployed.

### Key Vault

With the following secrets created:

- azure-registry-server
- azure-registry-username
- azure-registry-password
- azure-batch-location
- azure-batch-accountName
- azure-batch-accountKey
- azure-storage-accountName
- azure_storage_accountName
- azure-storage-accountKey
- azure_storage_accountKey

### Container Registry

- Nextflow image built and pushed to registry.
- Ubuntu image built and pushed to registry.
- Admin user enabled and keys stored in Key Vault.

### Storage Account

- Storage blob with 'batch' container created.
- Storage files with 'batchsmb' share created.
- Keys enabled and stored in Key Vault.

### Managed Identities

- Create a Managed Identity for Batch Account 'batchmid'.
- Create a Managed Identity for nextflow Container Instance(s) 'nextflowmid'

### Batch Account

- Pool allocation mode set to "Batch service".
- Connected to above Storage Account using User Assigned Managed Identity.
- Keys enabled and stored in Key Vault.

### Container Instance

- Configured using the nextflow image.
- Read access granted in Key Vault using User Assigned Managed Identity.
- Override the "AZ_KEY_VAULT_NAME" environment variable with the name of the Key Vault resource.
- Override the "AZURE_CLIENT_ID" environment variable with the client id of the nextflowmid Managed Identity resource.
- (optional) Override the CMD with one similar to that below to execute a different Nextflow pipeline.
    ``` bash
    cd /.nextflow && \
    ./nxfutil -c "https://<...>/nextflow.config" \
        -p "https://<...>/pipeline.nf" \
        -a "https://<...>/parameters.json"
    ``` 

## Usage

Once deployed the Container Instance will start and execute the default command of `./nxfutil` or that provided when the Container Instance was deployed.

Once called, nxfutil will download the default or provided Nextflow files and parse the "nextflow.config" to create a list of secrets it will need to retrieve from Key Vault. At this time, it will also expand and replace any `exParams` parameters with their values (also retrieved from Key Vault).

Once the config file has been parsed nxfutil will show the resultant Nextflow config by running `nextflow config` and will finally offload to nextflow by running `nextflow run` specifying the pipeline and parameters files.

After the `nextflow` command completes successfully the Contain Instance will stop. 

It is intended that the next iteration of nxfutil will integrate with Nextflow Tower so that launched jobs can be monitored remotely. It is also possible that the utility will be re-written using nodejs, dotNet and/or Rust.

## Lifecycle

A new nextflow Container Instance is needed for each different Nextflow pipeline, parameters and configuration combination.

Once a Container Instance executes and terminates it can be safely deleted unless the same job is to be dispatched again (using the same Nextflow pipeline, parameters and configuration files).

The quickest way to dispatched different Nextflow jobs is to use the Azure Cli to create a new nextflow Container Instance.

``` bash
az_subId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
az_rgName="myRgName"
az_kvName="myKvName"
az_crName="myCrName"
az_midClientId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

nxf_configUri="https://raw.githubusercontent.com/axgonz/azure-nextflow/main/nextflow/pipelines/nextflow.config"
nxf_pipelineUri="https://raw.githubusercontent.com/axgonz/azure-nextflow/main/nextflow/pipelines/helloWorld/pipeline.nf"
nxf_parametersUri="https://raw.githubusercontent.com/axgonz/azure-nextflow/main/nextflow/pipelines/helloWorld/parameters.json"

az container create -g $az_rgName \
    --name nextflow1 \
    --image "$az_crName.azurecr.io/default/nextflow:latest" \
    --cpu 1 \
    --memory 1 \
    --restart-policy Never \
    --environment-variables AZ_KEY_VAULT_NAME=$az_kvName AZURE_CLIENT_ID=$az_midClientId \
    --assign-identity "/subscriptions/$az_subId/resourcegroups/$az_rgName/providers/Microsoft.ManagedIdentity/userAssignedIdentities/nextflowmid" \
    --command-line "/bin/bash -c 'cd /.nextflow && ./nxfutil -c $nxf_configUri -p $nxf_pipelineUri -a $nxf_parametersUri'"
```

> N.B. The point-in-time Docker build of the nextflow image used in this sample is available on Docker Hub. To use it replace `--image ...` in the command above with `--image algonz/nextflow:latest`