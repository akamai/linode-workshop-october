Linode LevelUp Workshop
======================

# About

Package of template files, examples, and illustrations for the Linode Level Up Workshop Exercise.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files for starting an application on an LKE cluster.

### Exercise Diagram

## Step by Step Instructions

### Overview

The scenario is written to approximate deployment of a resillient, node-based API endpoint, that can provide both synchronous (via HTTP) and async (via a variety of pub/sub protocols) services to end users.

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in envinroment setup.

3. A Linode Kubernetes Engine (LKE) Cluster, provisioned via terraform.

4. Deploying a Node container to the cluster, and exposing an HTTP service from this deployment mapped to the node API endpoint. 


### Build a Secure Shell Linode
![shell](https://user-images.githubusercontent.com/19197357/184126449-454162f9-142f-47e6-ab73-3f1da5e5f456.png)

We'll first create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from. 

1. Create a Linode account
 - Goto https://login.linode.com/signup
 - Enter you akamai email address, a user name, and password
 - (Akamai employees get $100 per month of free services)

2. Login to Linode Cloud Manager
 - https://login.linode.com/login
3. Select "Create Linode"
4. Select "Marketplace"
5. Click the "Secure Your Server" Marketplace image. 
6. Scroll down and complete the following steps:
 - Limited sudo user
 - Sudo password
 - Ssh key
 - No Advanced options are required

7. Select the Debian 11 image type for Select an Image
8. Select a Region.
9. Select the Shared CPU 1GB "Nanode" plan.
10. Enter a root password.
11. Click Create Linode.

12. Once your Linode is running, login to it's shell (either using the web-based LISH console from Linode Cloud Manager, or via your SSH client of choice).



### Install and Run git 

Next step is to install git, and pull this repository to the Secure Shell Linode. The repository includes terraform and kubernetes configuration files that we'll need for subsequent steps.

1. Install git via the SSH or LISH shell-

```
sudo apt-get install git
```
2. Pull down this repository to the Linode machine-

```
git init && git pull https://github.com/akamai/linode-failover-workshop
```

### Install Terraform 

Next step is to install Terraform. Run the below commands from the Linode shell-
```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```
```
 wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
```
(Note:  This command may return what appears to be garbage to the terminal screen, but it does work.  Press `ctrl`-C to get your command line prompt back).
 
```
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```
```
sudo apt update && sudo apt-get install terraform
```

### Provision LKE Cluster using Terraform
![tf](https://user-images.githubusercontent.com/19197357/184130473-91c36dfc-072b-43f7-882b-07407d7f2266.png)

Next, we build the LKE cluster, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).
 - Click on your user name at the top right of the screen
 - Select API Tokens
 - Click Create a Personal Access Token
 - Be sure to copy and save the token value


2. From the Linode shell, set the TF_VAR_token env variable to the API token value. This will allow terraform to use the Linode API for infrastructure provisioning.
```
export TF_VAR_token=[api token value]
```
3. Initialize the Linode terraform provider-
```
terraform init 
```
4. Next, we'll use the supplied terraform files to provision the LKE clusters. First, run the "terraform plan" command to view the plan prior to deployment-
```
terraform plan \
 -var-file="terraform.tfvars"
 ```
 5. Run "terraform apply" to deploy the plan to Linode and build your LKE clusters-
 ```
 terraform apply \
 -var-file="terraform.tfvars"
 ```
Once deployment is complete, you should see a LKE clusters within the "Kubernetes" section of your Linode Cloud Manager account.

### Deploy Containers to LKE 
![k8](https://user-images.githubusercontent.com/19197357/184130510-08d983b6-109c-4bdb-b50c-db97fec3571d.png)

Next step is to use kubectl to deploy the node endpoints to the LKE cluster. 

1. Install kubectl via the below commands from the Linode shell-
```
sudo apt-get update && sudo apt-get install -y ca-certificates curl && sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update && sudo apt-get install -y kubectl
```
2. Extract the needed kubeconfig from each cluster into a yaml file from the terraform output.
```
 export KUBE_VAR=`terraform output kubeconfig_1` && echo $KUBE_VAR | base64 -di > lke-cluster-config1.yaml
```
```
 export KUBE_VAR=`terraform output kubeconfig_2` && echo $KUBE_VAR | base64 -di > lke-cluster-config2.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config1.yaml
```
4. You can now use kubectl to manage the LKE cluster. Enter the below command to view a list of clusters, and view which cluster is currently being managed.
```
kubectl config get-contexts
```
5. Deploy an application to the the LKE cluster, using the deployment.yaml file included in this repository.
```
kubectl create -f deployment.yaml
```
6. Next, we need to set our certificate and private key values as kubeconfig secrets. This will allow us to enable TLS on our LKE cluster. 

NOTE: For ease of the workshop, the certificate and key are included in the repository. This is not a recommended practice.
```
kubectl create secret tls mqtttest --cert cert.pem --key key.pem
```
7. Deploy the service.yaml included in the repository via kubectl to allow inbound traffic.
```
kubectl create -f service.yaml
```
8. Validate that the service is running, and obtain it's external IP address.
```
kubectl get services -A
```
This command output should show a node-api deployment, with an external (Internet-routable, non-RFC1918) IP address. Make note of this external IP address as it represents the ingress point to your cluster application.

### Summary of Linode Provisioning 

With the work above done, you've successfully setup a kubernetes cluster in a linode region, and deployed an endpoint application to it. 


