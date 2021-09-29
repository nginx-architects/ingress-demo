# Nginx Plus Ingress Controller Demo
This codebase provides everything you need to deploy the Nginx Plus Ingress Controller for Kubernetes in your cloud provider of choice. It includes deployments that route for some basic apps, configure SSL, create dashboards for metrics, and configure autoscaling (coming soon).

This is meant to be a guide for quickly standing up specific components and use cases for the Nginx Plus Ingress Controller.  If a lot of this is new to you and you would prefer for walk through these components step-by-step, you can follow the labs in our [Nginx Ingress Workshop](https://github.com/jasonwilliams14/nginx-ingress-workshops)

## Manual Configuration
Charts that map to use cases and their install instructions are located in the sub-folders of this repo.  Read the table of contents summaries below and drill into the folders that are relevant to your use-cases.

Table of Contents
* Cloud Deployments -- Select a cloud provider to deploy your cluster into.  
    * [AWS](aws-deployment/)
    * Azure (coming soon)
    * GCP (coming soon)
* [SSL Config](ssl/) -- Generate a self-signed cert on a wild-card domain so the secret can be used for any service in this repo.
* [Nginx Plus Dashboard](dashboard/) -- Charts for a service and route that map to the Nginx Plus Dashboard running on a port on each NIC
* Backend Deployments and Routes -- Charts that create backend services and routes that NIC will use to direct traffic
    * [Cafe](cafe/) -- Deployment and VirtualServer ingress for a basic cafe app with two endpoints
    * [Bar](bar/) -- Deployment and VirtualServer ingress for a basic bar app with three endpoints
* [Metrics and Monitoring](monitoring/) -- Instructions for installing Prometheus and Grafana to collect log metrics from the cluster and visually graph them 