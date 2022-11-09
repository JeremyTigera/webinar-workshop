# Tigera Calico Cloud - Security Workshop - Simplified version
In depth version: https://github.com/tigera-solutions/calicocloud-aks-workshop

## Implementing Container Security

This repository was created for the webinar about container security with Calico Cloud.
For this event, we will use a Kubernetes cluster and we will deploy two cloud native applications, some policies, alerts, reports and talk about features.

## Preparing the Cluster

Create a cluster in any supported platform such as AKS, EKS, GKE, RKE, Open Shift, Tanzu and so on.
When the cluster is up and run, we will connect it to Calico Cloud by running a simple command.
To create your own 14-day trial account, click on the link below:
https://www.calicocloud.io/home

We also offer an On-Premises version of Calico Cloud called Calico Enterprise.
Both versions are similar.

Now the cluster is connected to Calico Cloud, we will start by deploying a simple cloud native application.
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
![image](https://user-images.githubusercontent.com/101111449/168093686-bf8634b2-3b3d-41dd-bec2-fc0ae8bdf196.png)

You will notice that this application has been developped to use a Frontend, Backend, Logging and 2 Microservices and they have been designed to use the full potential of Calico Cloud's Network Policies. With that label schema, we can deploy Storefront application in a zone based architecture to enforce Microsegmentation and Zero-Trust.

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

#### Platform Tier

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

#### Security Tier

We will create a quarantine policy in the security tier, which is higher than the product and platform.
Meaning it has a higher priority.

```
kubectl apply -f https://raw.githubusercontent.com/JeremyTigera/webinar-workshop/main/security.quarantine
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
We already have a quarantine policy that is based on the quarantine label.
```
kubectl label pods attacker-pod quarantine=true
```


## Introduction to Threat Feeds
### Global Thread Feeds

Create the FeodoTracker globalThreatFeed:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/feodo-tracker.yaml
```

Verify the GlobalNetworkSet is configured correctly:
```
kubectl get globalnetworksets threatfeed.feodo-tracker -o yaml
```

Applies to anything that IS NOT listed with the namespace selector = 'acme'
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/block-feodo.yaml
```

Create a Default-Deny in the 'Default' namespace:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/default-deny.yaml
```

### Anonymization Attacks
Create the threat feed for KNOWN-MALWARE which we can then block with network policy:
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/malware-ipfeed.yaml
```
Create the threat feed for Tor Bulk Exit Nodes:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/tor-exit-feed.yaml
```
Additionally, feeds can be checked using following command:

```
kubectl get globalthreatfeeds
```
As you can see from the below example, it's making a pull request from a dynamic feed and labelling it - so we have a static selector for the feed:
```
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: vpn-ejr
spec:
  pull:
    http:
      url: https://raw.githubusercontent.com/n1g3ld0uglas/EuroEKSClusterCC/main/ejrfeed.txt
  globalNetworkSet:
    labels:
      threatfeed: vpn-ejr
```

## Deploy the Boutique Store Application
```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml
```
We also offer a test application for Kubernetes-specific network policies, the commands can be found in the more in depth repository below:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/test.yaml
```
https://github.com/tigera-solutions/calicocloud-aks-workshop/blob/main/modules/pod-access-controls.md

#### Block the test application
Deny the frontend pod traffic:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```
Allow the frontend pod traffic:
```
kubectl delete -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```
#### Introduce segmented policies
Deploy policies for the Boutique application:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/boutique-policies.yaml
```
Deploy policies for the K8 test application:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/test-app.yaml
```
![image](https://user-images.githubusercontent.com/101111449/200919633-a72aaf31-1fba-414e-8959-c0255bbba6d6.png)

## Alerts and Reports
Documentation for creating GlobalAlert custom resources:
https://docs.calicocloud.io/reference/resources/globalalert

### Alerts
#### Alert on NetworkSet changes:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/networksets.yaml
```

#### Alert on suspicious DNS Access requests:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/dns-access.yaml
```

#### Alert on lateral access to a specific namespace:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/
```
![image](https://user-images.githubusercontent.com/101111449/200922829-8b544df8-2751-4e96-9503-c282632743af.png)

### Reports

#### Generate a ``` CIS Benchmark```  report: <br/>

```   
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroAKSWorkshopCC/main/cis.yaml
```

#### Generate an ```Inventory```  report
```  
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroAKSWorkshopCC/main/inventory.yaml
```

#### Generate a ```Network Access```  report:
``` 
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/EuroAKSWorkshopCC/main/network.yaml 
```
<br/>
https://docs.calicocloud.io/compliance/overview

![compliance-reporting](https://user-images.githubusercontent.com/82048393/144321272-d6303cde-18b3-434a-b2ff-d45c6d9ccece.png)


Confirm your three reports are running as expected:
```
kubectl get globalreports
```

Ensure that the compliance-benchmarker is running, and that the cis-benchmark report type is installed:
```
kubectl get -n tigera-compliance daemonset compliance-benchmarker
kubectl get globalreporttype cis-benchmark
```


In the following example, we use a GlobalReport with CIS benchmark fields to schedule a ```DAILY``` report <br/>
https://docs.calicocloud.io/compliance/compliance-reports-cis
```
apiVersion: projectcalico.org/v3
kind: GlobalReport
metadata:
  name: daily-cis-results
  labels:
    deployment: production
spec:
  reportType: cis-benchmark
  schedule: 0 0 * * *
  cis:
    highThreshold: 100
    medThreshold: 50
    includeUnscoredTests: true
    numFailedTests: 5
    resultsFilters:
    - benchmarkSelection: { kubernetesVersion: "1.13" }
      exclude: ["1.1.4", "1.2.5"]
```

<img width="1458" alt="Screenshot 2022-05-05 at 14 42 11" src="https://user-images.githubusercontent.com/82048393/166936183-96695c59-8172-4597-a92e-bd424a748696.png">


The report is scheduled to run at midnight of the next day (in UTC), and the benchmark items ```1.1.4```  and  ```1.2.5``` will be omitted from the results.
<br/>
<br/>
Set it to ```schedule: 0 * * * *```  if you wish to generate ```HOURLY``` reports.

## Scan image registries for known vulnerabilities
Image Assurances helps operators identify vulnerabilities in workloads they deploy to Kubernetes clusters. <br/>
A vulnerability is a weakness in an application (for example, a design flaw or an implementation bug) that allows attackers to cause additional harm.<br/>
<br/>
The Common Vulnerabilities and Exposures (CVE) system provides publicly known information and security vulnerabilities and exposures. <br/>
Known vulnerabilities are identified by a unique CVE ID, based on the year it was reported (for example, CVE-2021-44228). <br/>
https://docs.calicocloud.io/image-assurance/scan-image-registries

<img width="1781" alt="Screenshot 2022-05-05 at 14 30 21" src="https://user-images.githubusercontent.com/82048393/166934933-bb61e30b-8e89-49a4-8f8e-6e51eb48aacd.png">

## More features and use case scenarios in the Tigera Official Github Repo in the following link
https://github.com/tigera-solutions/calicocloud-aks-workshop

#Thanks
