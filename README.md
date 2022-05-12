# Webinar/Workshop - Calico Entreprise/Calico Cloud

This repository was created for the webinar about Calico Entreprise.
For this event, we will use two GKE Clusters, one to introduce Calico Entreprise and a second one to deploy policies, alerts, audits etc.

## Preparing the Cluster

Create a cluster in any supported platform such as AKS, EKS, GKE, RKE, Open Shift, Tanzu and so on.
When the cluster is ready, the easiest and fastest way to manage your policies is to connect this cluster to Calico Cloud as you can create your own trial account here and will have 14 days free:
https://www.calicocloud.io/home

There is no real difference between Calico Cloud and Entreprise, one is a SaaS and the second one is On-Premise.
If you have to install Calico Enterprise, you will have to contact a Sales Representative to get a license key.

Now the cluster is installed with Calico Enterprise, we will need to deploy an application.
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
![image](https://user-images.githubusercontent.com/101111449/168093686-bf8634b2-3b3d-41dd-bec2-fc0ae8bdf196.png)

You will notice that this application has been developped to use a Frontend, Backend, Logging and 2 Microservices and they have been designed to use the full potential of Calico Entreprise's Network Policies. With that label schema, we can deploy Storefront application in a zone based architecture to enforce Microsegmentation and Zero-Trust.

