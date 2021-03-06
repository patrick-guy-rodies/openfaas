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

## Pre-requisites

1. A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled

1. The kubectl command-line tool installed on your local machine and configured to connect to your cluster. You can read more about installing kubectl in the official documentation.

1. The wget command-line utility installed on your local machine. You can install wget using the package manager built into your operating system.
Once you have these components set up, you’re ready to begin with this guide.

### Creating Domain name

Using a registar such as https://my.freenom.com/ create a domain. For this example, we will be using pgr095.tk domain. As you can see I have used Maersk UID.

### Use Repo for tutorial
1. Clone the [patrickguyrodies/k8cluster](bitbucket.org:patrickguyrodies/openfaas.git) repo and cd into the root of the repo.
You must run two commands: git submodule init to initialize your local configuration file, and git submodule update to fetch all the data from that project and check out the appropriate commit listed in your superproject:

### Install the faas-cli
The CLI is also available on brew for MacOS users, however it may lag behind by a few releases:


                $ brew install faas-cli

### Deploy using kubectl and plain YAML

1. Use Submodule

                $ git submodule add https://github.com/openfaas/faas-netes

Remember that submodule will need to be initialise using these two commands:  You must run two commands: git submodule init to initialize your local configuration file, and git submodule update to fetch all the data from that project and check out the appropriate commit listed in your project.

2. Deploy the whole stack

                $ kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

Create a password for the gateway

                # generate a random password
                PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

                kubectl -n openfaas create secret generic basic-auth \
                --from-literal=basic-auth-user=admin \
                --from-literal=basic-auth-password="$PASSWORD"

Deploy OpenFaaS

                $ cd faas-netes && kubectl apply -f ./yaml

Set your OPENFAAS_URL, if using a NodePort this may be 127.0.0.1:31112.

If you're using a remote cluster, or you're not sure then you can also port-forward the gateway to your machine for this step.


                $ kubectl port-forward svc/gateway -n openfaas 31112:8080 &

Now log in:

                $ export OPENFAAS_URL=http://127.0.0.1:31112

                $ echo -n $PASSWORD | faas-cli login --password-stdin

### Setting Up Dummy Backend Services
Before we deploy the Ingress Controller, we’ll first create and roll out one dummy echo Service to which we’ll route external traffic using the Ingress. The echo Services will run the [hashicorp/http-echo](https://hub.docker.com/r/hashicorp/http-echo/) web server container, which returns a page containing a text string passed in when the web server is launched. To learn more about http-echo, consult its [GitHub Repo] (https://github.com/hashicorp/http-echo), and to learn more about Kubernetes Services, consult Services from the official Kubernetes docs.

Check echo1.yaml file inside repo
                $ nano echo1.yaml

Create the Kubernetes resources using kubectl apply with the -f flag, specifying the file you just saved as a parameter:

                $ kubectl apply -f echo1.yaml

You should see the following output:

                Output
                service/echo1 created
                deployment.apps/echo1 created

Verify that the Service started correctly by confirming that it has a ClusterIP, the internal IP on which the Service is exposed:

                $ kubectl -n openfaas get svc echo1

You should see the following output:

                Output
                NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
                echo1     ClusterIP   10.245.222.129   <none>        80/TCP    60s

This indicates that the echo1 Service is now available internally at 10.245.222.129 on port 80. It will forward traffic to containerPort 5678 on the Pods it selects.

Now that the echo1 Service is up and running, repeat this process for the echo2 Service.
> Service and Deployment are installed inside openfaas namespace

### Setting Up the Kubernetes Nginx Ingress Controller


We’ll begin by first creating the Kubernetes resources required by the Nginx Ingress Controller. These consist of ConfigMaps containing the Controller’s configuration, Role-based Access Control (RBAC) Roles to grant the Controller access to the Kubernetes API, and the actual Ingress Controller Deployment which uses v0.34.0 of the Nginx Ingress Controller image. To see a full list of these required resources, consult the manifest from the Kubernetes Nginx Ingress Controller’s GitHub repo.

To create these mandatory resources, use kubectl apply and the -f flag to specify the manifest file hosted on GitHub:

                $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.0/deploy/static/provider/cloud/deploy.yaml

Confirm that the Ingress Controller Pods have started

                $ kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx

Check that the Azure LoadBalancer has been created

                $ kubectl get svc --namespace=ingress-nginx

You should see

                Output
                NAME            TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
                ingress-nginx   LoadBalancer   10.245.247.67   203.0.113.0   80:32486/TCP,443:32096/TCP   20h

We can now point our DNS records at this external Load Balancer and create some Ingress Resources to implement traffic routing rules.

### Create the Ingress Resource

Let’s begin by creating a minimal Ingress Resource to route traffic directed at a given subdomain to a corresponding backend Service.

In this guide, we’ll use the test domain pgr095.tk. You should substitute this with the domain name you own.

We’ll first create a simple rule to route traffic directed at echo1.pgr095.tk to the echo1 backend service.

Begin by opening up a file called echo_ingress.yaml in your favorite editor:

                $nano echo_ingress.yaml

You can now create the Ingress using kubectl:

                $ kubectl apply -f echo_ingress.yaml
You’ll see the following output confirming the Ingress creation:

                Output
                ingress.extensions/echo-ingress created

To test the Ingress, navigate to your DNS management service and create A records for echo1.pgr095.tk pointing to the Azure Load Balancer’s external IP. The Load Balancer’s external IP is the external IP address for the ingress-nginx Service, which we fetched in the previous step. 

Once you’ve created the necessary echo1.pgr095.tk DNS records, you can test the Ingress Controller and Resource you’ve created using the curl command line utility.

From your local machine, curl the echo1 Service:

                $ curl echo1.example.com

You should get the following response from the echo1 service:

                Output
                echo1

This confirms that your request to echo1.pgr095.tk is being correctly routed through the Nginx ingress to the echo1 backend Service.

### Installing and Configuring Cert-Manager

Before we install cert-manager, we’ll first create a Namespace for it to run in:

                $ kubectl create namespace cert-manager

Next, we’ll install cert-manager and its Custom Resource Definitions (CRDs) like Issuers and ClusterIssuers. Do this by applying the manifest directly from the cert-manager GitHub repository :


                $ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.2/cert-manager.yaml
                

Verify our installation by checking cert-manager Namespace for running pods:

                $ kubectl get pods -n cert-manager

Output

                NAME                                       READY   STATUS    RESTARTS   AGE
                cert-manager-749df5b4f8-94nvn              1/1     Running   0          71s
                cert-manager-cainjector-67b7c65dff-x2zfs   1/1     Running   0          71s
                cert-manager-webhook-7d5d8f856b-mfdtd      1/1     Running   0          71s

Before we begin issuing certificates for our Ingress hosts, we need to create an Issuer, which specifies the certificate authority from which signed x509 certificates can be obtained. In this guide, we’ll use the Let’s Encrypt certificate authority, which provides free TLS certificates and offers both a staging server for testing your certificate configuration, and a production server for rolling out verifiable TLS certificates.

Let’s create a test Issuer to make sure the certificate provisioning mechanism is functioning correctly. Open a file named staging_issuer.yaml in your favorite text editor:

                $ nano staging_issuer.yaml

Roll out the ClusterIssuer using kubectl:

                $ kubectl create -f staging_issuer.yaml

You should see the following output:

Output

                clusterissuer.cert-manager.io/letsencrypt-staging created

Now that we’ve created our Let’s Encrypt staging Issuer, we’re ready to modify the Ingress Resource we created above and enable TLS encryption for the echo1.pgr095.tk paths. Uncomment tls lines.

                $ kubectl apply -f echo_ingress.yaml

Output
                ingress.networking.k8s.io/echo-ingress configured

You can use kubectl describe to track the state of the Ingress changes you’ve just applied:

                $ kubectl -n openfaas describe ingress

Output
                Events:
                Type    Reason             Age                  From                      Message
                ----    ------             ----                 ----                      -------
                Normal  UPDATE             100s (x2 over 118m)  nginx-ingress-controller  Ingress openfaas/echo-ingress
                Normal  CreateCertificate  100s                 cert-manager              Successfully created Certificate "echo-tls"

Once certificate created, you can run an additional describe to confirm
                $ kubectl -n openfaas describe certificate

Output
                Status:
                Conditions:
                    Last Transition Time:  2020-07-16T12:29:54Z
                    Message:               Certificate is up to date and has not expired
                    Reason:                Ready
                    Status:                True
                    Type:                  Ready
                Not After:               2020-10-14T11:29:52Z
                Events:
                Type    Reason        Age   From          Message
                ----    ------        ----  ----          -------
                Normal  GeneratedKey  2m    cert-manager  Generated a new private key
                Normal  Requested     2m    cert-manager  Created new CertificateRequest resource "echo-tls-1761831726"
                Normal  Issued        94s   cert-manager  Certificate issued successfully

that HTTPS has successfully been enabled, but the certificate cannot be verified as it’s a fake temporary certificate issued by the Let’s Encrypt staging server.

Lets check our headers to see if we are redirected to port 443 and if the certificate is in place.

                $ wget --save-headers -O- echo1.pgr095.tk

Now that we’ve tested that everything works using this temporary fake certificate, we can roll out production certificates for our host.

### Rolling Out Production Issuer

In this step we’ll modify the procedure used to provision staging certificates, and generate a valid, verifiable production certificate for our Ingress hosts.

To begin, we’ll first create a production certificate ClusterIssuer.

Open a file called prod_issuer.yaml in your favorite editor to check the syntax:

                $nano prod-issuer.yaml

Note the different ACME server URL, and the letsencrypt-prod secret key name.

Now, roll out this Issuer using kubectl:

                $ kubectl create -f prod-issuer.yaml

You should see the following output:

Output

                clusterissuer.cert-manager.io/letsencrypt-prod created

Update echo_ingress.yaml to use this new Issuer:

                $ nano echo_ingress.yaml

Make the following change to the file by updating the ClusterIssuer name to letsencrypt-prod.

Once you’re satisfied with your changes, save and close the file.

Roll out the changes using kubectl apply:

                $ kubectl apply -f echo_ingress.yaml

Wait a couple of minutes for the Let’s Encrypt production server to issue the certificate. You can track its progress using kubectl describe on the certificate object:

                $ kubectl describe certificate echo-tls

Output

                Status:
                Conditions:
                    Last Transition Time:  2020-07-16T12:49:59Z
                    Message:               Certificate is up to date and has not expired
                    Reason:                Ready
                    Status:                True
                    Type:                  Ready
                Not After:               2020-10-14T11:49:59Z
                Events:
                Type    Reason        Age                From          Message
                ----    ------        ----               ----          -------
                Normal  GeneratedKey  20m                cert-manager  Generated a new private key
                Normal  Requested     20m                cert-manager  Created new CertificateRequest resource "echo-tls-1761831726"
                Normal  Requested     36s                cert-manager  Created new CertificateRequest resource "echo-tls-862082456"
                Normal  Issued        10s (x2 over 20m)  cert-manager  Certificate issued successfully
                patrick-maersk-macbook:openfaas patrickrodies$ 

Check your HTTPS either using curl or using openfaas portal by invoking the certinfo function (this function is from the openfaas store).

### Rolling Out Production Issuer for openfaas dashboard

Please check openfaas_ingress.yaml file. The only difference will be the name of secret to openfaas-tls

Once check run follow command 

                $ kubectl create -f openfaas_ingress.yaml

Output

                Status:
                Conditions:
                    Last Transition Time:  2020-07-16T13:47:12Z
                    Message:               Certificate is up to date and has not expired
                    Reason:                Ready
                    Status:                True
                    Type:                  Ready
                Not After:               2020-10-14T12:47:11Z
                Events:
                Type    Reason        Age   From          Message
                ----    ------        ----  ----          -------
                Normal  GeneratedKey  33s   cert-manager  Generated a new private key
                Normal  Requested     33s   cert-manager  Created new CertificateRequest resource "openfaas-tls-3421371503"
                Normal  Issued        5s    cert-manager  Certificate issued successfully

Check new certificate

                $ kubectl -n openfaas describe certificate openfaas-tls

