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

To confirm your rogue pod was successfully deployed to the 'default' namespace, run the below command:

```
kubectl get pods -n default
```

After you are done evaluating your network policies in with the rogue pod, you can remove it by running:

```
kubectl delete -f https://installer.calicocloud.io/rogue-demo.yaml
```

# Creating a zone-based architecture:
One of the most widely adopted deployment models with traditional firewalls is using a zone-based architecture. This
involves putting the frontend of an application in a DMZ, business logic services in Trusted zone, and our backend data
store in Restricted - all with controls on how zones can communicate with each other. For our storefront application, it
would look something like the following:


# Start with a Demilitarized Zone (DMZ):
The goal of a DMZ is to add an extra layer of security to an organization's local area network. 
A protected and monitored network node that faces outside the internal network can access what is exposed in the DMZ, while the rest of the organization's network is safe behind a firewall.

```
cat << EOF > dmz.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.dmz
  namespace: storefront
spec:
  tier: default
  order: 0
  selector: fw-zone == "dmz"
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: type == "public"
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: fw-zone == "trusted"||app == "logging"
    - action: Deny
      source: {}
      destination: {}
  types:
    - Ingress
    - Egress
EOF
```

```
kubectl apply -f dmz.yaml
```

# After the DMZ, we need a Trusted Zone
The trusted zone represents a group of network addresses from which the Personal firewall allows some inbound traffic using default settings.

```
cat << EOF > trusted.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.trusted
  namespace: storefront
spec:
  tier: default
  order: 10
  selector: fw-zone == "trusted"
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: fw-zone == "dmz"
      destination: {}
    - action: Allow
      source:
        selector: fw-zone == "trusted"
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: fw-zone == "restricted"
    - action: Deny
      source: {}
      destination: {}
  types:
    - Ingress
    - Egress
EOF
```

```
kubectl apply -f trusted.yaml
```

# Finally, we configure the Restricted Zone
A restricted zone supports functions to which access must be strictly controlled; direct access from an uncontrolled network should not be permitted. In a large enterprise, several network zones might be designated as restricted.

```
cat << EOF > restricted.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.restricted
  namespace: storefront
spec:
  tier: default
  order: 20
  selector: fw-zone == "restricted"
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: fw-zone == "trusted"
      destination: {}
    - action: Allow
      source:
        selector: fw-zone == "restricted"
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination: {}
  types:
    - Ingress
    - Egress
EOF
```

```
kubectl apply -f restricted.yaml
```

# Subscribing to a malicious threatfeed
Project Calico simplifies the process of dynamic threat feed subscription via a Calico construct known as 'NetworkSets'.
A network set resource represents an arbitrary set of IP subnetworks/CIDRs, allowing it to be matched by Calico policy. 
Network sets are useful for applying policy to traffic coming from (or going to) external, non-Calico, networks.

```
cat << EOF > threat-feed.yaml
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: feodo-tracker
spec:
  content: IPSet
  pull:
    http:
      url: https://feodotracker.abuse.ch/downloads/ipblocklist.txt
EOF
```

As always, don't forget to add the feed to your cluster.

```
kubectl apply -f threat-feed.yaml
```

This pulls updates using the default period of once per day. 
See the Global Resource Threat Feed API for all configuration options:
https://docs.tigera.io/reference/resources/globalthreatfeed

# Build a policy based on the threat feed

We will start by creating a tier called 'security'.
Notice how the below 'block-fedo' policy is related to the 'security' tier - name: security.block-feodo

```
cat << EOF > feodo-policy.yaml
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
EOF
```

```
kubectl apply -f feodo-policy.yaml
```

# Verify policy on test workload
We will verify the policy from the test workload that we created earlier.
Exec into a pod 

```
kubectl apply -f https://k8s.io/examples/application/shell-demo.yaml
```

Get a shell to the running pod

```
kubectl exec --stdin --tty shell-demo -- /bin/bash
```

Install the ping utility:

```
apt update && apt install iputils-ping
```

Ping a known safe IP (like 8.8.8.8, Google’s public DNS server)

```
ping 8.8.8.8
```

Open the FEODO tracker list and choose an IP on the list to ping.
https://feodotracker.abuse.ch/downloads/ipblocklist.txt

You should not get connectivity, and the pings will show up as denied traffic in the flow logs.

# PCI Whitelist

This policy ensures that only pods with PCI-labeled service accounts can talk to each other. 
For traffic that does not involve a PCI-labeled service account, we use the Pass action to “pass” this traffic to the next tier of Calico network policies

```
cat << EOF > pci-whitelist.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.pci-whitelist
spec:
  tier: security
  order: 255
  selector: ''
  namespaceSelector: ''
  serviceAccountSelector: PCI == "true"
  ingress:
    - action: Deny
      source:
        serviceAccounts:
          names: []
          selector: PCI != "true"
      destination:
        serviceAccounts:
          names: []
          selector: PCI == "true"
  egress:
    - action: Pass
      source: {}
      destination:
        selector: k8s-app == "kube-dns"||has(dns.operator.openshift.io/daemonset-dns)
    - action: Pass
      source: {}
      destination:
        selector: type == "public"
    - action: Deny
      source:
        serviceAccounts:
          names: []
          selector: PCI == "true"
      destination:
        serviceAccounts:
          names: []
          selector: PCI != "true"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```

```
kubectl apply -f pci-whitelist.yaml
```

# Compliance Reporting
In this section we will walk through a quick example of how to use Calico Enterprise to produce dynamic compliance
reports that allow you to assess the state of compliance that is in lock step with your CI/CD pipeline

```
cat << EOF > daily-cis-results.yaml
apiVersion: projectcalico.org/v3
kind: GlobalReport
metadata:
 name: daily-cis-results
 labels:
 deployment: production
spec:
 reportType: cis-benchmark
 schedule: 0 * * * *
 cis:
 highThreshold: 100
 medThreshold: 50
 includeUnscoredTests: true
 numFailedTests: 5
 resultsFilters:
 - benchmarkSelection: { kubernetesVersion: “1.15” }
 exclude: [“1.1.4”, “1.2.5”]
 EOF
 ```
 
 ```
kubectl apply -f daily-cis-results.yaml
```
 
CIS benchmarks are best practices for the secure configuration of a target system - in our case Kubnernetes. 
Calico Cloud supports a number of GlobalReport types that can be used for continuous compliance, and CIS benchmarks is one of them.
