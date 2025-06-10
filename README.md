Observability for Jupyterhub in Kubernetes with Elastic APM
======================

# About

Package of template files and instructions on:
1. how to set up a kubernetes cluster on Linode with Terraform
2. deploy jupyterhub via helm, kubectl
3. deploy the elastic otel collector
   
# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.


## Step by Step Instructions


### Build a Secure Shell Linode
We'll first create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from. 

1. Login to Linode Cloud Manager, Select "Create Linode," and choose the "Secure Your Server" Marketplace image. 
2. Within the setup template for "Secure Your Server," select the Ubuntu 20.04 LTS image type. 
3. Once your Linode is running, login to it's shell (either using the web-based LISH console from Linode Cloud Manager, or via your SSH client of choice).

### Install and Run git 

Next step is to init git, and pull this repository to the Secure Shell Linode. The repository includes terraform and kubernetes configuration files that we'll need for subsequent steps.

1. Pull down this repository to the Linode machine-

```
git init && git pull https://github.com/abeaudin/jupyterhub-o11y
```

### Install Terraform 

Next step is to install Terraform. Run the below commands from the Linode shell-

```
 curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
 echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com noble main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
 sudo apt update && sudo apt-get install terraform
  ```

### Provision LKE Cluster using Terraform

Next, we build the LKE cluster, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).

2. From the Linode shell, set the TF_VAR_token env variable to the API token value. This will allow terraform to use the Linode API for infrastructure provisioning.
```
export TF_VAR_token=[api token value]
```
3. Initialize the Linode terraform provider-
```
terraform init 
```
4. Next, we'll use the supplied terraform files to provision the LKE cluster. First, run the "terraform plan" command to view the plan prior to deployment-
```
terraform plan \
 -var-file="terraform.tfvars"
 ```
 5. Run "terraform apply" to deploy the plan to Linode and build your LKE cluster-
 ```
 terraform apply \
 -var-file="terraform.tfvars"
 ```
Once deployment is complete, you should see an LKE cluster within the "Kubernetes" section of your Linode Cloud Manager account.

### Deploy Containers to LKE 

Next step is to use kubectl to deploy the elastic stack to the LKE cluster. 

1. Install kubectl via the below commands from the Linode shell-
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg 
```
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update && sudo apt-get install -y kubectl
```
2. Extract the needed kubeconfig from each cluster into a yaml file from the terraform output.
```
 export KUBE_VAR=`terraform output kubeconfig_1` && echo $KUBE_VAR | base64 -di > lke-cluster-config.yaml
```
```
chmod 400 lke-cluster-config.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config.yaml
```
4. Install helm
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
```
```
chmod 700 get_helm.sh
```
```
./get_helm.sh
```

5. Configure helm with the juptyerhub repo
```
helm repo add jupyterhub https://hub.jupyter.org/helm-chart
```
```
helm repo update
```


helm repo add jupyterhub https://hub.jupyter.org/helm-chart
helm repo update
helm upgrade --cleanup-on-fail --install my-jupyter jupyterhub/jupyterhub 
kubectl get pods
helm repo add open-telemetry 'https://open-telemetry.github.io/opentelemetry-helm-charts' --force-update
kubectl create namespace opentelemetry-operator-system
kubectl create secret generic elastic-secret-otel   --namespace opentelemetry-operator-system   --from-literal=elastic_otlp_endpoint='https://ELASTIC-HOST-NAME-From-KIBANA.elastic.cloud:443'   --from-literal=elastic_api_key='ELASTIC-API-KEY-From-KIBANA'
helm upgrade --install opentelemetry-kube-stack open-telemetry/opentelemetry-kube-stack   --namespace opentelemetry-operator-system   --values 'https://raw.githubusercontent.com/elastic/elastic-agent/refs/tags/v9.0.2/deploy/helm/edot-collector/kube-stack/managed_otlp/values.yaml'   --version '0.3.9'

kubectl annotate namespace default instrumentation.opentelemetry.io/inject-python="opentelemetry-operator-system/elastic-instrumentation"
kubectl rollout restart deployment/user-scheduler
kubectl rollout restart deployment/hub
kubectl rollout restart deployment/proxy
