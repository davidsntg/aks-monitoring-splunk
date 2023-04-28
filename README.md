# Lab - AKS Monitoring with Splunk

This LAB is for POC purpose only!

- [Lab - AKS Monitoring with Splunk](#lab---aks-monitoring-with-splunk)
- [Infrastructure Provisionning](#infrastructure-provisionning)
  * [Azure - Create Splunk VM](#azure---create-splunk-vm)
  * [Install Splunk](#install-splunk)
  * [Configure Splunk](#configure-splunk)
  * [Azure - Create AKS cluster](#azure---create-aks-cluster)
  * [Azure - Create Event Hub Namespace & Event Hub](#azure---create-event-hub-namespace---event-hub)
- [Configuration - Control Plane Logs & Audit logs to Splunk](#configuration---control-plane-logs---audit-logs-to-splunk)
  * [Azure - Configure AKS Diagnostic Settings](#azure---configure-aks-diagnostic-settings)
  * [Splunk - Configure Splunk Add-on for Microsoft Cloud Services](#splunk---configure-splunk-add-on-for-microsoft-cloud-services)
- [Configuration - Applications Logs & Metrics to Splunk](#configuration---applications-logs---metrics-to-splunk)
  * [Splunk - Create HTTP Event Collector](#splunk---create-http-event-collector)
  * [Splunk - Enable HTTP Event Collector](#splunk---enable-http-event-collector)
  * [AKS - Install collectorforkubernetes solution](#aks---install-collectorforkubernetes-solution)
  * [Configure collectorforkubernetes solution](#configure-collectorforkubernetes-solution)

# Infrastructure Provisionning

## Azure - Create Splunk VM

* Create Splunk VM:
```bash
location="westeurope"
rg_name="splunk-rg"

vnet_name="vnet"
vnet_prefix="10.0.0.0/24"
subnet_name="snet-splunk"
subnet_prefix="10.0.0.0/24"

vm_name="splunk-vm"
vm_size="Standard_B8ms"
ssh_admin_user="davidsantiago"
ssh_admin_password="Microsoft=1Microsoft=1"
splunk_fqdn_prefix="demo-splunk-david"
splunk_fqdn="${splunk_fqdn_prefix}.${location}.cloudapp.azure.com"

az group create -n "$rg_name" -l "$location"

az network vnet create -g "$rg_name" -n "$vnet_name" --address-prefixes "$vnet_prefix" --subnet-name "$subnet_name" --subnet-prefixes "$subnet_prefix" --location "$location"

az vm create -g "$rg_name" -n "$vm_name" --image Ubuntu2204 --admin-username "$ssh_admin_user" --admin-password "$ssh_admin_password" --vnet-name "$vnet_name" --subnet "$subnet_name" --public-ip-sku Standard --public-ip-address-dns-name "$splunk_fqdn_prefix" --nsg-rule SSH --size "$vm_size"

az network nsg rule create -g "$rg_name" --nsg-name "${vm_name}NSG" -n "Splunk" --priority 1010 --source-address-prefixes '*' --destination-address-prefixes '*' --destination-port-ranges 8000 8089 9997 8088 9998 514 9999 1514 --access Allow --protocol Tcp --description "Splunk ports"

echo "Connect to Splunk VM: ssh ${ssh_admin_user}@${splunk_fqdn}"
```

## Install Splunk

* Create and account on Splunk Enteprise website and copy Linux .deb installer download link:

![image](https://user-images.githubusercontent.com/87186004/235093298-0c6f6776-d0ea-4d24-b1cf-8fc509c656f1.png)


* Connect to Splunk VM in SSH and execute below commands:

```bash
sudo su -
apt update -y && apt upgrade -y
apt install wget apt-transport-https gnupg2 -y 
wget https://download.splunk.com/products/splunk/releases/9.0.4.1/linux/splunk-9.0.4.1-419ad9369127-linux-2.6-amd64.deb
dpkg -i splunk-9.0.4.1-419ad9369127-linux-2.6-amd64.deb
/opt/splunk/bin/splunk enable boot-start --accept-license 
# Enter login & password
systemctl start splunk
```

## Configure Splunk

* Sign In to http://${splunk_fqdn}:8000 

![image](https://user-images.githubusercontent.com/87186004/235092744-45ee66d8-a45f-4f27-9cc3-a1a400c0ff14.png)

* Install "Splunk Add-on for Microsoft Cloud Services":

![image](https://user-images.githubusercontent.com/87186004/235093619-0ff44edc-89d5-4bc2-8e50-9576fc12093e.png)

* Install "Monitoring Kubernetes - Metrics and Log Forwarding":

![image](https://user-images.githubusercontent.com/87186004/235093878-29f6a75e-edb6-4388-9c9d-2fa06d6d7c49.png)

## Azure - Create AKS cluster

* Create AKS cluster:
```bash
AKS_RESOURCE_GROUP="aks-splunk-rg"
AKS_REGION="westeurope"
AKS_NAME="aks-demo-splunk"

az group create -n "$AKS_RESOURCE_GROUP" -l "$AKS_REGION"
az aks create -g "$AKS_RESOURCE_GROUP" -n "$AKS_NAME" --enable-managed-identity --node-count 1 --enable-addons monitoring --enable-msi-auth-for-monitoring  --generate-ssh-keys
```

## Azure - Create Event Hub Namespace & Event Hub

* Create Event Hub Namespace & Event Hub:
```bash
EH_RESOURCE_GROUP="eventhub-rg"
EH_REGION="westeurope"
EH_NS_NAME="eh-ns-monitoring-$RANDOM"
EH_NAME="eh-monitoring-$RANDOM"
az group create -n "$EH_RESOURCE_GROUP" -l "$EH_REGION"
az eventhubs namespace create --name "$EH_NS_NAME" --resource-group "$EH_RESOURCE_GROUP" -l "$AKS_REGION"
az eventhubs eventhub create --name "$EH_NAME" --resource-group "$EH_RESOURCE_GROUP" --namespace-name "$EH_NS_NAME"
```

# Configuration - Control Plane Logs & Audit logs to Splunk

## Azure - Configure AKS Diagnostic Settings

* Configure AKS cluster to send all logs & metrics to Event Hub:

![image](https://user-images.githubusercontent.com/87186004/235100424-c19b7798-7fd5-407b-b147-af9a0936d718.png)

## Splunk - Configure Splunk Add-on for Microsoft Cloud Services

* Create an Azure Service Principal:
```bash
az ad sp create-for-rbac --name "AZ_SP_SPLUNK" --skip-assignment
```

* Assign `Azure Event Hubs Data Receiver` role to Service Principal on Event Hub Namespace:
```bash
az role assignment create --role "Azure Event Hubs Data Receiver" --assignee "4653eade-c757-43b9-xxxx-97855b3aab40" --scope "/subscriptions/2c49b441-xxxx-xxxx-xxxx-cd28d472544d/resourceGroups/eventhub-rg/providers/Microsoft.EventHub/namespaces/eh-ns-monitoring-1602"
```

* Configure SP in Splunk Add-On for Microsoft Cloud Services
![image](https://user-images.githubusercontent.com/87186004/235101396-fa5770b6-3c17-4de9-b6e5-2507002956ce.png)

* Add Event Hub as an input of the Add-On:

![image](https://user-images.githubusercontent.com/87186004/235102487-325e4caf-54c9-4570-aa19-bc541871993e.png)

* Check logs are available:

![image](https://user-images.githubusercontent.com/87186004/235103018-16845ee0-8861-423c-b046-5bacac03289d.png)

# Configuration - Applications Logs & Metrics to Splunk

## Splunk - Create HTTP Event Collector

* Add a new HTTP Event Collector:

![image](https://user-images.githubusercontent.com/87186004/235104098-e7902a52-fdcb-4f2b-a5b2-e25cdc7bf6f2.png)

* Give it a name:

![image](https://user-images.githubusercontent.com/87186004/235104203-61666010-2ec1-4d4e-b1d2-a7bb4e9b9251.png)

* Leave Input Settings blank:

![image](https://user-images.githubusercontent.com/87186004/235104250-89d5c886-0d7b-47a9-be4d-db05e7839629.png)

* Submit:

![image](https://user-images.githubusercontent.com/87186004/235104295-faa47f25-d6bb-4485-9fcb-65da0f6cb0b2.png)

## Splunk - Enable HTTP Event Collector

* Enable HTTP Event Collector:

![image](https://user-images.githubusercontent.com/87186004/235104627-7f468949-a57b-4c59-9255-6ea525f54bad.png)

* Go to Global Settings:

![image](https://user-images.githubusercontent.com/87186004/235104756-ed33ea40-47f6-4b2b-aa37-2b93d7d9eca8.png)

* Enable & Save:

![image](https://user-images.githubusercontent.com/87186004/235104847-36775242-dce2-43e8-8b1e-ca7118ea3b23.png)

## AKS - Install collectorforkubernetes solution

Installation guide is available here: https://www.outcoldsolutions.com/docs/monitoring-kubernetes/v5/installation/

To achieve this question, you must request a trial license to outcold solutions.

## Configure collectorforkubernetes solution

* Download yaml file:
```bash
wget https://www.outcoldsolutions.com/docs/monitoring-kubernetes/v5/configuration/1.24/collectorforkubernetes.yaml
```

* Edit it, only below fields with your license / token / url:
```bash
vi collectorforkubernetes.yaml
```

```yaml
[general]

acceptLicense = true

license = Q0gwSDJISUVQUjBMNjoxMDA6MTY4NDU3Nzg2Mjo0.HZXmL7Gd8j57C4PJZpQuKu4zC2MCN+43+Y4jHQ.JkmlmuUbrP/o2OHclNzpVhMvsWoWZCGubGdRRg

fields.kubernetes_cluster = aks-demo-splunk

...

# Splunk output
[output.splunk]

# Splunk HTTP Event Collector url
url = http://demo-splunk-david.westeurope.cloudapp.azure.com:8088/services/collector/event/1.0

# Splunk HTTP Event Collector Token
token = 7451d8c8-799e-4f28-ab67-09ea3b0c28c8

# Allow invalid SSL server certificate
insecure = true
```

* Apply the configuration to AKS cluster:

```bash
az account set --subscription 2c49b441-xxxx-xxxx-xxxx-cd28d472544d
az aks get-credentials --resource-group aks-splunk-rg --name aks-demo-splunk
kubectl apply -f ./collectorforkubernetes.yaml
```

* Check Daemon sets:

```bash
kubectl get ds -n collectorforkubernetes
NAME                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
collectorforkubernetes          1         1         1       1            1           <none>          91s
collectorforkubernetes-master   0         0         0       0            0           <none>          91s
```






