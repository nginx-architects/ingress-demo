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

At this point you should be able to get the nodes with kubectl:
```
$ kubectl get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-34-166.ec2.internal   Ready    <none>   35d   v1.21.2-eks-55daa9d
ip-192-168-5-162.ec2.internal    Ready    <none>   35d   v1.21.2-eks-55daa9d
ip-192-168-79-128.ec2.internal   Ready    <none>   35d   v1.21.2-eks-55daa9d
```

## Access & Authorization

Creates an OIDC IdP for the cluster:

`eksctl utils associate-iam-oidc-provider --region=<eks-cluster-region> --cluster=<eks-cluster-name> --approve`

Creates IAM service account that gets tied to a namespace and the cluster, and attaches an IAM policy for monitoring usage of the NIC Plus image and authorizing its deployment into the cluster:

`eksctl create iamserviceaccount --name nginx-ingress --namespace nginx-ingress --cluster <eks-cluster-name> --region <eks-cluster-region> --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage --approve`

You can change the service account name and namespace but you would then need to propagate those changes to some of the upcoming charts.  If you use `nginx-ingress` all of the charts that reference the service account or namespace will work as is.

Edit lines 104/105 in `rbac.yaml` to match your service account name and namespace if you changed them from the default `nginx-ingress`

Creates the ClusterRole and ClusterRoleBinding for authorizing the service account you just created:

`kubectl apply -f rbac.yaml`

## Create Elastic Load Balancer & Allow Traffic

Creates an ELB in AWS with a LoadBalancer type Service routes all HTTP/HTTPS traffic (80/443) from the ELB to the `nginx-ingress` app which we haven't created yet:

`kubectl apply -f loadbalancer-aws-elb.yaml`

Applies some Nginx config values including what traffic is allowed to the LoadBalancer.  Needs to be applied after the LoadBalancer Service is created allow traffic through it:

`kubectl apply -f nginx-config.yaml`

To retrieve the public endpoint for the service we just created:

`kubectl get svc -n nginx-ingress`

## Push Image to ECR & Deploy NIC Plus

Now we've deployed everything we need except the ingress controller itself.  We need to publish the NIC Plus image to ECR and then reference it in the `nginx-plus-ingress.yaml` before we can apply the chart.

There are ![a couple ways](https://docs.nginx.com/nginx-ingress-controller/installation/using-the-jwt-token-docker-secret/) to publish the licensed image.  For AWS, the easiest is to ![Subscribe to NGINX Ingress Controller](https://aws.amazon.com/marketplace/pp/prodview-fx3faxl7zqeau?qid=1626138210561&sr=0-2&ref_=srh_res_product_title) in the AWS Marketplace.

Once you go through the `subscribe` and `configuration` (Select EKSDelivery for latest NICversion) menus, the `launch` menu will have a `View container image details` link that will open a modal.  Follow these instructions to log in to the registry with your docker client and pull the image.

Copy the image name into Line 21 of `nginx-plus-ingress.yaml`.  It will look something like this: `709825985650.dkr.ecr.us-east-1.amazonaws.com/nginx/nginx-plus-ingress:2.0.2`

If you used a different Service Account name than the default `nginx-ingress`, change it on Line 19.

Now we're ready to create the ingress controller DaemonSet:

`kubectl apply -f nginx-plus-ingress.yaml`

You should see one ingress running for each of your nodes:

```
$ kubectl get pods -n nginx-ingress -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP              NODE
nginx-ingress-2bwzf   1/1     Running   0          6m25s   192.168.14.61   ip-192-168-5-162.ec2.internal 
nginx-ingress-gxq77   1/1     Running   0          6m6s    192.168.72.94   ip-192-168-79-128.ec2.internal
nginx-ingress-vnqg2   1/1     Running   0          6m13s   192.168.36.16   ip-192-168-34-166.ec2.internal
```

Run `kubectl describe ds -n nginx-ingress` if you see something different than the above to diagnose the issue.

To test, get the EXTERNAL-IP for the ELB LoadBalancer service that we created earlier:

```
$ kubectl get svc -n nginx-ingress
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                         PORT(S)                      AGE
dashboard-svc   ClusterIP      10.100.85.179   <none>                              9000/TCP                     29d
nginx-ingress   LoadBalancer   10.100.233.58   <xxx>.<region>.elb.amazonaws.com   80:32108/TCP,443:31476/TCP   29d
```

Visit the ELB endpoint in your browser and you should be taken to the default 404 page hosted by your new ingress controller.

## Hosts File

To determine the IP Addresses of the ELB to add to your hosts file on your computer, run the following:

`dig <xxx>.<region>.elb.amazonaws.com`

You should get a list of IP addresses in the `;; ANSWER SECTION:` that you can map to the appropriate endpoints.

This example lists all the endpoints used in other sections of this repo:

```
# EKS ELB IP
1.1.1.2 1.1.2.3 1.2.3.4 cafe.ingress.demo dashboard.ingress.demo bar.ingress.demo grafana.ingress.demo prometheus.ingress.demo
```