# Getting started with Calico Cloud
This quickstart guide for Calico Network Policy and Security options for Kubernetes clusters is just an extension of Tigera's documentation for their Software-as-a-Service offering 'Calico Cloud' - https://docs.calicocloud.io/get-started/quickstart

This guide also assumes that you have already created a supported Kubernetes cluster, have installed Project Calico on the master node, and have started a trial of Project Calico.

At the time of writing this post, you can easily hook-up your existing cluster to Calico Cloud by running the cURL command provided to you by the team at Tigera. 
The command should look similar to the below example:

```
curl -s https://installer.calicocloud.io/XYZ_your_business_install.sh | bash
```

# Accessing the web user interface
If you're struggling to log into the web user interface, you'll need to generate a base64 encoded token for user authentication.
Create a user in the desired namespace. Once created, we can assign the necessary permissions.

```
kubectl create sa nigel -n default
```

The clusterrolebinding name is a just a generic descriptive name for the rolebinding. However, the clusterrole name specifies Calico Cloud UI permissions.
The role 'tigera-network-admin' provides full access to the Calico Cloud Manager, allows the user to create and modify Calico Cloud resources, as well as providing super-user access for Kibana, including Elasticsearch usernamespace is the service account’s namespace.
Finally, the serviceaccount is the service account that the permissions are being associated with.

```
kubectl create clusterrolebinding nigel-access --clusterrole tigera-network-admin --serviceaccount default:nigel
```

Next, get the token from the service account. Using the running example of a service account named 'nigel' in the default namespace.

```
kubectl get secret $(kubectl get serviceaccount nigel -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

If you need a user with limited permissions, we can apply clusterrole=tigera-network-ui the the new service account.
This provides a basic user with access to Calico Cloud Manager and Kibana. The user can List/view Calico Cloud policy, Kubernetes policy, and tier resources - as well as Listing/viewing logs within Kibana:
https://docs.calicocloud.io/install/user-management

# Deploying a Rogue Pod for instant threat visibility

If your cluster does not have applications, you can use the following storefront application. This .YAML file creates a bunch of NameSpaces, ServiceAccounts, Deployments and Services responsible for real-world visibility into complex environment. This 'Storefront' namespace contains the standard microservices, frontend, backend and logging components we would expect in a cloud-native architecture.

```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Check which pods are running within the newly added 'Storefront' namespace. 
Showing the labels associated with the pods will help us later with policy configuration.

```
kubectl get pod -n storefront --show-labels
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

This pulls updates using the default period of once per day. 
See the Global Resource Threat Feed API for all configuration options:
https://docs.tigera.io/reference/resources/globalthreatfeed

# Build a policy based on the threat feed

We will start by creating a tier called 'security'.
Notice how the below 'block-fedo' policy is related to the 'security' tier - name: security.block-feodo

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.block-feodo
spec:
  tier: security
  order: 210
  selector: projectcalico.org/namespace != "acme"
  namespaceSelector: ''
  serviceAccountSelector: ''
  egress:
    - action: Deny
      source: {}
      destination:
        selector: threatfeed == "feodo"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Egress
```
# Test the threat feed for yourself
We will now apply a GlobalNetworkPolicy that blocks the test workload from connecting to any IPs in the threat feed.
Create a file called 'block-feodo.yaml' with the following contents:

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default.block-feodo
spec:
  tier: default
  selector: docs.tigera.io/tutorial == 'threat-feed'
  types:
  - Egress
  egress:
  - action: Deny
    destination:
      selector: docs.tigera.io/threat/feed == 'feodo'
  - action: Allow
```

Apply this policy to the cluster

```
kubectl apply -f block-feodo.yaml
```
