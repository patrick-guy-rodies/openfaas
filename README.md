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

1. A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled

1. The kubectl command-line tool installed on your local machine and configured to connect to your cluster. You can read more about installing kubectl in the official documentation.

1. The wget command-line utility installed on your local machine. You can install wget using the package manager built into your operating system.
Once you have these components set up, you’re ready to begin with this guide.

### Creating Domain name

Using a registar such as https://my.freenom.com/ create a domain. For this example, we will be using pgr095.tk domain. As you can see I have used Maersk UID.

### Use Repo for tutorial
1. Clone the [patrickguyrodies/k8cluster](bitbucket.org:patrickguyrodies/openfaas.git) repo and cd into the root of the repo.

### Install the faas-cli
The CLI is also available on brew for MacOS users, however it may lag behind by a few releases:


                $brew install faas-cli


### Setting Up Dummy Backend Services
Before we deploy the Ingress Controller, we’ll first create and roll out two dummy echo Services to which we’ll route external traffic using the Ingress. The echo Services will run the [hashicorp/http-echo](https://hub.docker.com/r/hashicorp/http-echo/) web server container, which returns a page containing a text string passed in when the web server is launched. To learn more about http-echo, consult its [GitHub Repo] (https://github.com/hashicorp/http-echo), and to learn more about Kubernetes Services, consult Services from the official Kubernetes docs.

Check echo1.yaml file inside repo
                $ nano echo1.yaml

Create the Kubernetes resources using kubectl apply with the -f flag, specifying the file you just saved as a parameter:

                $ kubctl apply -f echo1.yaml

You should see the following output:

                Output
                service/echo1 created
                deployment.apps/echo1 created

Verify that the Service started correctly by confirming that it has a ClusterIP, the internal IP on which the Service is exposed:

                $ kubectl get svc echo1

You should see the following output:

                Output
                NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
                echo1     ClusterIP   10.245.222.129   <none>        80/TCP    60s

This indicates that the echo1 Service is now available internally at 10.245.222.129 on port 80. It will forward traffic to containerPort 5678 on the Pods it selects.

Now that the echo1 Service is up and running, repeat this process for the echo2 Service.
> Both backend services are installed under Default namespaces. We are using them as examples for our ingress.

### Setting Up the Kubernetes Nginx Ingress Controller


