# Lab - AKS Monitoring with Splunk

## Azure - Create Splunk VM

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

Create and account on Splunk Enteprise website and copy Linux .deb installer download link:

![image](https://user-images.githubusercontent.com/87186004/235093298-0c6f6776-d0ea-4d24-b1cf-8fc509c656f1.png)


Connect to Splunk VM in SSH and execute below commands:

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

Sign In to http://${splunk_fqdn}:8000 

![image](https://user-images.githubusercontent.com/87186004/235092744-45ee66d8-a45f-4f27-9cc3-a1a400c0ff14.png)

Install "Splunk Add-on for Microsoft Cloud Services":

![image](https://user-images.githubusercontent.com/87186004/235093619-0ff44edc-89d5-4bc2-8e50-9576fc12093e.png)

Install "Monitoring Kubernetes - Metrics and Log Forwarding":

![image](https://user-images.githubusercontent.com/87186004/235093878-29f6a75e-edb6-4388-9c9d-2fa06d6d7c49.png)

## Azure - Create AKS cluster
