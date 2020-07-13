# OpenFaaS implementation with TLS

This repo contains Yaml file to install OpenFaaS with TLS. Certificate is from LetsEncrypt, Domain used is from freenom and DNS from Azure

## Key features

* LetsEncrypt certificate
* DNS Azure service
* Domain name from Free service called freenom
* No use of Helm - :-)


## Possible Improvements
* Disaster Recovery
* MakeFile

### Pre-requisites

1. Cluster on AKS

1. kubectl

### Creating Domain name


1. Clone the [patrickguyrodies/k8cluster](bitbucket.org:patrickguyrodies/k8cluster.git) repo and cd into the root of the repo.

1. Initialise state and Import stateful resources
    