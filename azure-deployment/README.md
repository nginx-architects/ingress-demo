# Azure AKS Cluster and NIC Deployment

## Client Prerequisites
* Docker client
* latest version of [kubectl](https://kubernetes.io/docs/tasks/tools/)
* latest version of the [Azure CLI (az)](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (EKS CLI to create the cluster)
* [Azure Subscription](https://docs.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli) with permissions to create resource groups, create/publish to azure's docker registry, and to create the AKS cluster

## Create Cluster
Creates a resource group for your cluster resources in the appropriate region:

`az group create --name <group-name> --location eastus`

Creates the AKS cluster and generates ssh keys for the VMs (this takes a bit):

`az aks create --resource-group <group-name> --name <cluster-name> --node-count 3 --generate-ssh-keys`

Adds Azure credentials and context to kubectl:
`az aks get-credentials --resource-group <group-name> --name <cluster-name>`

Get nodes with kubectl should look something like this:
```
$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-18086562-vmss000000   Ready    agent   23h   v1.20.9
aks-nodepool1-18086562-vmss000001   Ready    agent   23h   v1.20.9
aks-nodepool1-18086562-vmss000002   Ready    agent   23h   v1.20.9
```

To list other configured clusters:

`kubectl config get-contexts`

Use output to set the current kubeconfig to a different context:

`kubectl config use-context <cluster-name>`

To delete the AKS cluster you just delete the resource group that contains it:

`az group delete --name <group-name> --yes --no-wait`