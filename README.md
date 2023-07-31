# Azure Red Hat OpenShift Private cluster with Jump Host
The following guide is a basic tutorial to install an ARO private cluster with an additional 
jump host to connect to and verify the installation.

## Client prerequisites
Export the Azure environment configurations.
```
export GUID=<YOUR_GUID>
export CLIENT_ID=<YOUR_CLIENT_ID>
export PASSWORD=<YOUR_PASSWORD>
export TENANT=<YOUR_TENANT>
export SUBSCRIPTION=<YOUR_SUBSCRIPTION_ID>
```

Install the Azure CLI:
```
curl -L https://aka.ms/InstallAzureCli | bash
```

The following client tools must also be installed on the host used for the deployment:
- sshuttle (https://github.com/sshuttle/sshuttle for install instructions)
- OpenShift CLI (https://access.redhat.com/downloads/content/290 for download)

## Azure infrastructure prerequisites
### Subscription ID choice
Login to the subscription using the service principal:
```
az login --service-principal -u $CLIENT_ID -p $PASSWORD --tenant $TENANT
```

In case of multiple subscriptions, specify the correct subscription ID. 
```
az account set --subscription <SUBSCRIPTION ID>
```

### Register Resource Providers
Enable the mandatory providers to install the ARO cluster.
```
az provider register -n Microsoft.RedHatOpenShift --wait
az provider register -n Microsoft.Compute --wait
az provider register -n Microsoft.Network --wait
az provider register -n Microsoft.Storage --wait
```

### Create Resource Group
Create a dedicated Resource Group for the new ARO cluster.
```
LOCATION=westeurope             # the location of your cluster
RESOURCEGROUP="v4-$LOCATION"    # the name of the resource group where you want to create your cluster
CLUSTER=aro-cluster             # the name of your cluster

az group create --name $RESOURCEGROUP --location $LOCATION
```

### Create Vnet
Create a Vnet in the new Resource Group with the address prefix of choice.
```
az network vnet create --resource-group $RESOURCEGROUP --name aro-vnet --address-prefixes 10.0.0.0/22
```

### Create control-plane subnet
Create the control plane subnet in the new VNet. The address prefix must be smaller than the VNet prefix.
```
az network vnet subnet create   \
--resource-group $RESOURCEGROUP \
--vnet-name aro-vnet            \
--name master-subnet            \
--address-prefixes 10.0.0.0/24  \
--service-endpoints Microsoft.ContainerRegistry
```

### Create worker subnet
Create the worker subnet in the new VNet. The address prefix must be smaller than the VNet prefix.
```
az network vnet subnet create   \
--resource-group $RESOURCEGROUP \
--vnet-name aro-vnet            \
--name worker-subnet            \
--address-prefixes 10.0.1.0/24  \ 
--service-endpoints Microsoft.ContainerRegistry
```

### Disable private link endpoint policies
Disable subnet private endpoint policies on the master subnet. This is required to be able to connect and manage the cluster.
```
az network vnet subnet update   \
--name master-subnet            \
--resource-group $RESOURCEGROUP \
--vnet-name aro-vnet            \
--disable-private-link-service-network-policies true
```

## Cluster creation
### ARO private cluster creation
Create the private ARO cluster. Notice the `--apiserver-visibility Private` and `--ingress-visibility Private` flags.
The Pull Secret is not mandatory but strongly suggested. Download the pull secret from this [link](https://www.ibm.com/links?url=https%3A%2F%2Fcloud.redhat.com%2Fopenshift%2Finstall%2Fpull-secret) and put in a path of choice.  
The `--pull-secret` points at the pull secret file name (do not forget to prepend the file path with `@`).
```
az aro create                     \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER                 \
  --vnet aro-vnet                 \
  --master-subnet master-subnet   \
  --worker-subnet worker-subnet   \
  --apiserver-visibility Private  \
  --ingress-visibility Private    \
  --pull-secret @pull-secret.txt 
```

Wait for 35-40 minutes until the installation is completed. To view current ARO clusters:
```
az aro list
```

## Jump host configuration
Since we are creating a private cluster it is necessary to create a jump host to access the cluster API and Ingress.
This is a commodity to manage and configure the cluster and not necessary when the Vnet and Subnets are already correctly routed to the on-prem infrastructure (via VPN or ExpressRoute).

### Create jump host subnet
The jump host subnet is create in the same Vnet and Resource Group as the cluster. In this example it gets a /24 subnet.
```
az network vnet subnet create   \
--resource-group $RESOURCEGROUP \
--vnet-name aro-vnet            \
--name jump-subnet              \
--address-prefixes 10.0.2.0/24  \
--service-endpoints Microsoft.ContainerRegistry
```

### Create jump host VM
The jump host VM is create in the new subnet and a RHEL9 (not mandatory) system is used in the example.  
The jump host public ip address (see option `--public-ip-address`) is automatically created if not existing.  

```
SSH_PUBKEY=$HOME/.ssh/id_rsa.pub

az vm create --name jumphost                 \
    --resource-group $RESOURCEGROUP          \
    --ssh-key-values $SSH_PUBKEY             \
    --admin-username aro                     \
    --image "RedHat:RHEL:9_1:9.1.2022112113" \
    --subnet jump-subnet                     \
    --public-ip-address jumphost-ip          \
    --public-ip-sku Standard                 \
    --vnet-name aro-vnet
```

### Open tunnel connection
Once the jump host is up and running it is possible to grab its public IP and start the tunnel connection.
On a new terminal window, star the VPN tunnel. Remember to redefine the exported variables if necessary.
```
JUMP_IP=$(az vm list-ip-addresses -g $RESOURCEGROUP -n jumphost -o tsv \
--query '[].virtualMachine.network.publicIpAddresses[0].ipAddress')

sshuttle --dns -NHr "aro@${JUMP_IP}"  10.0.0.0/8
```

The subnet(s) passed as argument to `sshuttle` provides a list of subnets to route over the VPN. A 0/0 subnet has the command route everything through the VPN (very unsecure!).




### Connect to the cluster
It is now possible to connect to the ARO private cluster. First, obtain the API server endpoint.
```
APISERVER=$(az aro show              \
--name aro-cluster                   \
--resource-group $RESOURCEGROUP      \
-o tsv --query apiserverProfile.url)
```

Get the `kubeadmin` user password.
```
ADMINPW=$(az aro list-credentials    \
--name aro-cluster                   \
--resource-group $RESOURCEGROUP      \
--query kubeadminPassword            \
-o tsv)
```

Test the cluster login via API Server.
```
oc login $APISERVER --username kubeadmin --password ${ADMINPW}
```

To login via console run the following command:
``` 
az aro show                              \
    --name $CLUSTER                      \
    --resource-group $RESOURCEGROUP      \
    --query "consoleProfile.url" -o tsv
```

## Extras

### User Defined Routing (optional)
Typically, private clusters are created with a public IP address and load balancer. However, it is possible to enable User Defined Routing to prevent the creation of a public IP address for outbount connectivity. 
**This is a technical preview feature and is not supported in production environments.**

```
az feature register --namespace Microsoft.RedHatOpenShift --name UserDefinedRouting
```

For egress, the User Defined Routing option ensures that the newly created cluster has the egress lockdown feature enabled to allow you to secure outbound traffic from your new private cluster.

## References
This tutorial is derived from the following docs:
- https://learn.microsoft.com/en-us/azure/openshift/howto-create-private-cluster-4x
- https://mobb.ninja/docs/aro/private-cluster/
- https://linux.die.net/man/8/sshuttle
- https://learn.microsoft.com/en-us/azure/openshift/concepts-networking#networking-components
