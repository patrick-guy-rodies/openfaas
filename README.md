# Deployment for Infrastructure for implementing a cluster with default values

This repo contains all the code required to deploy & configure a AKS cluster latest version
This is using service principal for application and storage for keeping terraform state

## Key features

* AKS cluster of three nodes in Northern Region
* Linux OS with 30 gb disk


## Possible Improvements or Security issue
* Security key not encrypted, will need to use Vault 
* Disaster Recovery
* MakeFile

### Pre-requisites

1. terraform

1. az (Azure CLI)

1. kubectl

1. Storage setup using tstate

### Deploying

1. Clone the [patrickguyrodies/k8cluster](bitbucket.org:patrickguyrodies/k8cluster.git) repo and cd into the root of the repo.

1. Initialise state and Import stateful resources
    