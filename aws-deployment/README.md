# AWS Cluster and NIC Deployment
This section details how to install an EKS (Elastic Kubernetes Service) Cluster and Nginx Plus Ingress Controller on that cluster.

## Client Prerequisites
* Docker client
* latest version of [kubectl](https://kubernetes.io/docs/tasks/tools/)
* latest version of [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) (EKS CLI to create the cluster)
* AWS admin credentials set in `~/.aws/credentials`

## Architecture
This is what we will be deploying:

![AWS EKS Architecture](https://www.nginx.com/wp-content/uploads/2021/08/NIC-on-EKS-how-tested_topology.svg)

## Create & Configure Cluster
Creates an EKS cluster with the specified instance sizing in your default region. Can specify the region with `--region` flag:

`eksctl create cluster --instance-types=c5n.9xlarge`

Note: In `us-east-1` you will likely get an `UnsupportedAvailabilityZoneException`. If you do, the output will list other availability zones to try.copy the suggested zones and pass `--zones` flag: 

`eksctl create cluster --region=us-east-1 --zones=us-east-1a,us-east-1b,us-east-1d`

To get cluster names:

`eksctl get cluster`

Creates an OIDC IdP for the cluster:

`eksctl utils associate-iam-oidc-provider --region=<eks-cluster-region> --cluster=<eks-cluster-name> --approve`

Creates IAM service account for the Nginx Plus Ingress Controller that gets tied to a namespace and the cluster, and attaches an IAM policy for monitoring usage of the NIC Plus image and authorizing its deployment into the cluster:

`eksctl create iamserviceaccount --name nginx-ingress --namespace nginx-ingress --cluster <eks-cluster-name> --region <eks-cluster-region> --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage --approve`

You can change the service account name and namespace but you would then need to propagate those changes to some of the upcoming charts.  If you use `nginx-ingress` all of the charts that reference the service account or namespace will work as is.

At this point you should be able to get the nodes with kubectl:
```
$ kubectl get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-34-166.ec2.internal   Ready    <none>   35d   v1.21.2-eks-55daa9d
ip-192-168-5-162.ec2.internal    Ready    <none>   35d   v1.21.2-eks-55daa9d
ip-192-168-79-128.ec2.internal   Ready    <none>   35d   v1.21.2-eks-55daa9d
```

Edit lines 104/105 in `rbac.yaml` to match your service account name and namespace if you changed them from the default `nginx-ingress`

Creates the ClusterRole and ClusterRoleBinding for authorizing the service account you just created:

`kubectl apply -f rbac.yaml`

Creates an ELB in AWS with a LoadBalancer type Service routes all HTTP/HTTPS traffic (80/443) from the ELB to the `nginx-ingress` app which we haven't created yet:

`kubectl apply -f loadbalancer-aws-elb.yaml`

Applies some Nginx config values including what traffic is allowed to the LoadBalancer.  Needs to be applied after the LoadBalancer Service is created allow traffic through it:

`kubectl apply -f nginx-config.yaml`
