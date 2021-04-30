# Getting started with Calico Cloud
This quickstart guide for Calico Network Policy and Security options for Kubernetes clusters is just an extension of Tigera's documentation for their Software-as-a-Service offering 'Calico Cloud' - https://docs.calicocloud.io/get-started/quickstart

This guide also assumes that you have already created a supported Kubernetes cluster, have installed Project Calico on the master node, and have started a trial of Project Calico.

At the time of writing this post, you can easily hook-up your existing cluster to Calico Cloud by running the cURL command provided to you by the team at Tigera. 
The command should look similar to the below example:

```
curl -s https://installer.calicocloud.io/XYZ_your_business_install.sh | bash
```

# Deploying a Rogue Pod for instant threat visibility

If your cluster does not have applications, you can use the following storefront application. This .YAML file creates a bunch of NameSpaces, ServiceAccounts, Deployments and Services responsible for real-world visibility into complex environment. This 'Storefront' namespace contains the standard microservices, frontend, backend and logging components we would expect in a cloud-native architecture.

```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Run the following command to start a rogue workload to simulate a malicious actor probing for vulnerable or exposed services within the cluster:

```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml
```

After you are done evaluating your network policies in with the rogue pod, you can remove it by running:

```
kubectl delete -f https://installer.calicocloud.io/rogue-demo.yaml
```

# Subscribing to a malicious threatfeed
Project Calico simplifies the process of dynamic threat feed subscription via a Calico construct known as 'NetworkSets'.
A network set resource represents an arbitrary set of IP subnetworks/CIDRs, allowing it to be matched by Calico policy. 
Network sets are useful for applying policy to traffic coming from (or going to) external, non-Calico, networks.

```
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: feodo-tracker
spec:
  content: IPSet
  pull:
    http:
      url: https://feodotracker.abuse.ch/downloads/ipblocklist.txt
```

As always, don't forget to add the feed to your cluster.

```
kubectl apply -f feodo-tracker.yaml
```
