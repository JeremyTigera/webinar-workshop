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

![image](https://user-images.githubusercontent.com/101111449/168094979-fa4da469-19c7-4988-bc56-aabb5c1c6775.png)

With our Service Graph tool we can create a realtime topoly of this application and see clearly the communications between microservices.
On top of that we can see if some flows are blocked or allowed and let you know which policy is responsible.

## Implementing your security

First of all, with Calico Entreprise and Cloud we have an added feature for the Network Policies called Tiers.
You can create a zone based architecture to help creating scopes for your teams. They will be in charge of one or multiple zones, here below the documentation and a screenshot to visualise it better:
![image](https://user-images.githubusercontent.com/101111449/168097675-0400a123-e0a1-4eef-9f51-0b37679eee65.png)

https://docs.tigera.io/v3.13/security/tiered-policy#tiers-what-and-why

### Create Tiers

You can create a tier via yaml manifest or simply via the WebUI
We will create 3 tiers: Security, Platform and Product.
![image](https://user-images.githubusercontent.com/101111449/168101119-4cc0c8a1-402f-48bc-b593-c0202df0f2c2.png)

### Zone-Based Architecture

#### Product Tier

Using the label schema in the application Storefront we will be able to create policies for our microsegmentation project:

Create the DMZ Policy:
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/product.dmz
```
Create the Trusted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/product.trusted
``` 
Create the Restricted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/product.restricted
```

Confirm all policies are running:
```
kubectl get networkpolicies.p -n storefront -l projectcalico.org/tier=product
```

### Platform Tier

Allow Kube-DNS Traffic: 
We need to create the following policy within the ```tigera-security``` tier <br/>
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/platform.allowkubedns
```
Kubernetes best practise: Default Deny
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/product.default-deny-storefront
```

## Simulating an intruder

We will simulate an intruder was able to create a rogue pod within the Storefront namespace.

### Increase the Sync Rate

To get faster data;
``` 
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"dnsLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
```

### Introduce the rogue pod

Introduce the Rogue Application:
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml -n storefront
```

### Quarantine the Rogue Application: 

Instead of deleting the intruder, we will put it in quarantine for further investigation by the security team.
```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/tigera-security.quarantine
```
