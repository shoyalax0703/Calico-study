# Calico study

> To begin we will be installing Multipass. Multipass is a utility from Canonical that allows you to create Ubuntu VMs across a range of platforms in a uniform fashion. We recommend using the latest stable version of Multipass (version 1.5.0 at the time of writing). If any difficulty is encountered deploying the labs - please try this version of Multipass and let us know in the #academy slack channel that you encountered issues with a newer version. 

> Multipass installation instructions can be found here: https://multipass.run

> Note: If you're running on Windows, you must use the default Hyper-V option.  In addition, note that if you are running an old version of VMware Workstation/Player on Windows, you may find that VMware can not start VMs after using multipass due to its use of Hyper-V. If you experience this, please see the workaround instructions later in the "Managing Your Lab" module.

> Note 2: If you have issues with the Multipass VMs, consider disabling VPN clients that might be running on your system. We have seen scenarios where VPN clients can cause problems with VMs started by Multipass.

> Restart your workstation after installing Multipass and before installing the lab. It is essential you do not skip this step.

## Quick start
If you’re on Linux, Mac, or have access to a Bash shell on Windows you can follow these steps to get up and running quickly:

```
curl https://raw.githubusercontent.com/tigera/ccol1/main/control-init.yaml | multipass launch -n control -m 2048M 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node1-init.yaml | multipass launch -n node1 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node2-init.yaml | multipass launch -n node2 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/host1-init.yaml | multipass launch -n host1 20.04 --cloud-init -
```

## Starting the Instances
On some platforms, multipass requires you to start the VMs after they have been launched. We can do this by using the multipass start command.
```
multipass start --all
```
Throughout the deployments for the labs, the instances will reboot once provisioning is complete. As a result, you may have to wait a minute until the instance has fully provisioned. A quick way to check the current state of the cluster is to use the multipass list command.
```
multipass list
```
Example output:
<img width="908" alt="Screen Shot 2022-08-30 at 23 25 28" src="https://user-images.githubusercontent.com/66551005/187463110-686b43af-6aa2-4175-9003-579bc4a55f5c.png">



## Validating the Environment
To validate the lab has successfully started after all four instances we will enter the host1 shell:
```
multipass shell host1
```
Once you reach the command prompt of host1, run kubectl get nodes.
```
kubectl get nodes -A
```
Example output:
<img width="908" alt="Screen Shot 2022-08-30 at 23 26 00" src="https://user-images.githubusercontent.com/66551005/187463261-a3700400-813b-493d-90e0-52fafc43f45f.png">
  
Note the “NotReady” status. This is because we have not yet installed a CNI plugin to provide the networking.

The instance we will be using for the following labs will be host1 unless otherwise specified. Think of host1 as your primary entry point into the kubernetes ecosystem, with the other instances acting as the cluster in the cloud.

We will be using the Tigera Operator to install and configure Calico.
```
multipass shell host1
```
```
kubectl create -f https://docs.projectcalico.org/archive/v3.21/manifests/tigera-operator.yaml
```

Example output:
<img width="1350" alt="entrollersconfigurations crd projectcalico org created" src="https://user-images.githubusercontent.com/66551005/188762776-54ebe240-9210-4ea4-92aa-be5f40ef7581.png">

```
kubectl get pods -n tigera-operator
```
Example output:

<img width="1350" alt="STATUS RESTARTS" src="https://user-images.githubusercontent.com/66551005/188762765-5bf2f2c5-3735-4342-9a49-4e84828ad14f.png">

Installing Calico.
After the operator is in a Running state, we will configure an Installation kind for Calico, specifying the IP Pool that we would like below.
Note that throughout the course we make use of inline manifests (piping stdin to kubectl) to make it easier for you to follow what each manifest does. In most cases it would be a more normal practice to use a vanilla kubectl command with a manifest file (e.g. kubectl apply -f my-installation.yaml).  We recommend taking a minute to read through and make sure you understand the contents of each manifest we apply in this way throughout the rest of the course to get the most out of each example.

```
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    containerIPForwarding: Enabled
    ipPools:
    - cidr: 198.19.16.0/21
      natOutgoing: Enabled
      encapsulation: None
EOF
```

Following the configuration of the installation resource, Calico will begin deploying onto your cluster. This can be validated by running the following command:
```
kubectl get tigerastatus/calico
```
Example output:
<img width="1350" alt="Screen Shot 2022-08-31 at 6 55 27" src="https://user-images.githubusercontent.com/66551005/188762743-ef51d9d6-fc3b-4909-afa9-c7abff8d641e.png">

We can review the environment now by invoking
```
kubectl get pods -A
```
Example output:
<img width="1350" alt="Screen Shot 2022-08-31 at 6 56 41" src="https://user-images.githubusercontent.com/66551005/188762714-af60a7c4-d8e3-4a5d-994c-a58c7599698d.png">

Reviewing Calico pods.
Let's take a look at the Calico pods that have been installed by the operator.
```
kubectl get pods -n calico-system
```
Example output:
<img width="1350" alt="Screen Shot 2022-08-31 at 6 57 19" src="https://user-images.githubusercontent.com/66551005/188762700-35ea2f41-ebee-4131-b2c9-81087a09abfa.png">

From here we can see that there are different pods that are deployed.
1. calico-node: Calico-node runs on every Kubernetes cluster node as a DaemonSet. It is responsible for enforcing network policy, setting up routes on the nodes, plus managing any virtual interfaces for IPIP, VXLAN, or WireGuard.
2. calico-typha: Typha is as a stateful proxy for the Kubernetes API server. It's used by every calico-node pod to query and watch Kubernetes resources without putting excessive load on the Kubernetes API server.  The Tigera Operator automatically scales the number of Typha instances as the cluster size grows.
3. calico-kube-controllers: Runs a variety of Calico specific controllers that automate synchronization of resources. For example, when a Kubernetes node is deleted, it tidies up any IP addresses or other Calico resources associated with the node.

Reviewing Node Health.

Finally, we can review the health of our Kubernetes nodes by invoking the kubectl command.
```
kubectl get nodes -A
```
Example output:
<img width="1350" alt="Screen Shot 2022-08-31 at 6 58 06" src="https://user-images.githubusercontent.com/66551005/188762687-9104532c-b431-4338-990d-94ca90d2dc71.png">

Now we can see that our Kubernetes nodes have a status of Ready and are operational. Calico is now installed on your cluster and you may proceed to the next module: Installing the Sample Application.

## if you would like to stop studying during the course. 
Stop nodes.
```
exit
```
```
multipass stop --all
```
```
multipass list
```
Restart studying.
```
multipass start control
```
```
multipass start node1
```
```
multipass start node2
```
```
multipass start host1
```

Clean-up.
```
multipass stop --all
```
```
multipass delete --all
```
```
multipass purge
```

## Introduction to the Sample Application
For this lab, we will be deploying an application called "Yet Another Online Bank" (yaobank). The application will consist of 3 microservices.
![Introduction to the Sample Application](https://user-images.githubusercontent.com/66551005/188762676-dc611db8-069c-45d8-b7b9-9d17e910ebb7.png)

1. Customer (which provides a simple web GUI)
2. Summary (some middleware business logic)
3. Database (the persistent datastore for the bank)
All the Kubernetes resources (Deployments, Pods, Services, Service Accounts, etc) for Yaobank will all be created within the yaobank namespace.

To install yaobank into your kubernetes cluster, apply the following manifest:
```
kubectl apply -f https://raw.githubusercontent.com/tigera/ccol1/main/yaobank.yaml
```
Check the Deployment Status.
To validate that the application has been deployed into your cluster, we will check the rollout status of each of the microservices.

Check the customer microservice:
```
kubectl rollout status -n yaobank deployment/customer
```
Example output:
<img width="1142" alt="Screen Shot 2022-08-31 at 7 15 58" src="https://user-images.githubusercontent.com/66551005/188762663-d73a8f18-692f-464b-9933-87c02e10690c.png">

Check the summary microservice:
```
kubectl rollout status -n yaobank deployment/summary
```
Example output:
<img width="889" alt="deployment summary successfully rolled out" src="https://user-images.githubusercontent.com/66551005/188762652-ecb64d66-fa90-4372-8ba9-8610bb5fd6d1.png">

Check the database microservice:

```
kubectl rollout status -n yaobank deployment/database
```
Example output:
<img width="956" alt="kubectl rollout status" src="https://user-images.githubusercontent.com/66551005/188762642-0318c231-6d3e-4ff0-bb75-0847188a9ea9.png">

Access the Sample Application Web GUI.
Now we can browse to the service using the service’s NodePort. The NodePort exists on every node in the cluster. We’ll use the control node, but you get the exact same behavior connecting to any other node in the cluster.
```
curl 198.19.0.1:30180
```
Example output:
<img width="1102" alt="Screen Shot 2022-08-31 at 7 18 44" src="https://user-images.githubusercontent.com/66551005/188762626-d56c11e7-d8f8-4e7f-8069-d3c6b71ce1d9.png">

## Network Policy
![What is Network Policy](https://user-images.githubusercontent.com/66551005/188762606-0f452cd0-d88b-422a-8076-fadec7545f0f.png)

NetworkPolicyはPodに対してPod間の通信や外部のエンドポイントへの通信を制御するためのリソースだ。

why is it important? traditional firewalls struggle with dynamic nature of kubernetes network policy is label selector based -> inherently dynamic

Simulate a compromise.
To simulate a compromise of the customer pod we will exec into the pod and attempt to access the database directly from there.

### Enter the customer pod

First we will find the customer pod name and store it in an environment variable to simplify future commands.
```
CUSTOMER_POD=$(kubectl get pods -n yaobank -l app=customer -o name)
```

Note that the CUSTOMER_POD environment variable only exists within your current shell, so if you exit that shell you must set it again in your new shell using the same command as above.
Now we will exec into the customer pod, and run bash, to give us a command prompt within the pod:
```
kubectl exec -it $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```

### Access the database

From within the customer pod, we will now attempt to access the database directly, simulating an attack.  As the pod is not secured with NetworkPolicy, the attack will succeed and the balance of all users will be returned.
```
curl http://database:2379/v2/keys?recursive=true | python -m json.tool
```
Example output:
<img width="1231" alt="Screen Shot 2022-09-02 at 14 38 10" src="https://user-images.githubusercontent.com/66551005/188762586-45700414-9c0d-4ea1-9fd3-ac878959ad80.png">

Leaving the customer pod.
To return from the pod back to our original host command line shell, use the exit command:
```
exit
```

### Protect the database 
To protect the yaobank database, we will be using kubernetes policy.
Deploy the K8s Network Policy.
```
cat <<EOF | kubectl apply -f -
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: database-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: summary
    ports:
      - protocol: TCP
        port: 2379
  egress:
    - to: []
EOF
```
this network policy means allow ingress traffic only from summary pod in yaobook namespace to database pod in yaobook namespace.

Try the attack again.
Now let’s repeat the simulated attack.
Execute bash command in customer container
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```
Try to  access the database
```
curl --connect-timeout 3 http://database:2379/v2/keys?recursive=true
```
Example output:
<img width="1231" alt="Screen Shot 2022-09-02 at 14 56 40" src="https://user-images.githubusercontent.com/66551005/188762565-ddd34249-9db4-4514-bdac-60aa2f2119c9.png">

The curl will be blocked and return no data.
Then remember to exit the pod exec and return to the host terminal.
```
exit
```

Now you have seen how easy it is to protect a microservice using network policy, let’s look at how we switch the cluster to a default-deny behavior. This is a best practice that ensures that every new microservice deployed needs to have an accompanying network policy and no microservices are accidentally left wide open to attack.
You can do this on a per-namespace scope using Kubernetes Network Policy, but that requires each namespace to have its own default-deny policy, and relies on remembering to create a default-deny policy each time a new namespace is created. This is where Calico policy can be useful. Since Calico’s GlobalNetworkPolicy policies apply across all namespaces, you can write a single default-deny policy for the whole of your cluster.

### calicoctl

The calicoctl utility is used to create and manage Calico resource types, as well as allowing you to run a range of other Calico specific commands. In this specific case, we are creating a GlobalNetworkPolicy (GNP).
Deploy the network policy
```
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
  types:
  - Ingress
  - Egress
EOF
```

There a couple of things worth noting:
1. For this lab we’ve chosen to exclude kube-system and calico-system namespaces, since we don’t want the policy to impact the Kubernetes or Calico control planes. (This is a good best practice which avoids accidentally breaking the control planes, in case you have not already set up appropriate network policies and/or Calico failsafe port rules that allows control plane traffic. You can then separately write network policy for each control plane component.)
2. As the policy contains no rules, it doesn't actually matter what precedence it has, so we did not specify an order value. But for completeness of your learning, omitting the order field on a network policy means it has the lower precedence compared to any Calico network policy which does specify an order, or Kubernetes network policies which have an implicit order of 1000. 
Verify default deny is in place
In the “Protect the Database” step above, we only defined network policy for the Database. The rest of our pods should now be hitting default deny (for both ingress and egress) since there's no policy defined to say who they are allowed to talk to.
Let's try to see if basic connectivity works, e.g. DNS.
Execute bash command in customer container
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```
Now let’s try to do a DNS lookup.
```
dig www.google.com
```
That should fail, timing out after around 15s, because we've not written any policy to say DNS (or any other egress) is allowed.
After running the command, remember to exit from the kubectl exec of the customer pod.
```
exit
```

Lets update our default policy to allow DNS to the cluster-internal kube-dns service. 
```
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
  types:
  - Ingress
  - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - 53
EOF
```
Now we’ll attempt to do a DNS lookup again to confirm the policy now allows it.
Execute bash command in customer container.
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```
Try the DNS lookup again.
```
dig www.google.com
```
This should now succeed.
Example output:
<img width="747" alt="global options +cnd" src="https://user-images.githubusercontent.com/66551005/188762519-68455c40-73f1-4c3d-9772-ebb272fa2567.png">

After running the command, remember to exit from the kubectl exec of the customer pod by typing exit.
```
exit
```

We've now defined a default app policy across all of the cluster which results in default-deny behavior except for allowed DNS queries to kube-dns. We also defined a policy for the database earlier in the “Protect the Database” step. But we haven't yet defined policies for the customer or summary pods.
Verify we cannot access the Frontend
Try to access the Yaobank front end via the front end service’s NodePort:
```
curl --connect-timeout 3 198.19.0.1:30180
```
This should timeout because we haven't provided detailed policy for the customer and summary pods.
Create policy for the remaining pods
Let’s create the remaining policies for all the Yaobank services.
```
cat <<EOF | kubectl apply -f - 
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: customer-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: customer
  ingress:
    - ports:
      - protocol: TCP
        port: 80
  egress:
    - to: []
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: summary-policy
  namespace: yaobank
spec:
  podSelector:
    matchLabels:
      app: summary
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: customer
      ports:
      - protocol: TCP
        port: 80
  egress:
    - to:
      - podSelector:
          matchLabels:
            app: database
      ports:
      - protocol: TCP
        port: 2379
EOF
```
Verify everything is now working.
Now try accessing the Yaobank front end, it should work again. By defining the cluster wide default deny we've forced the user to follow best practices and define network policy for all the necessary microservices.
```
curl 198.19.0.1:30180
```
The resulting output should contain the 
following at the bottom:
```
  <body>
        <h1>Welcome to YAO Bank</h1>
        <h2>Name: Spike Curtis</h2>
        <h2>Balance: 2389.45</h2>
        <p><a href="/logout">Log Out >></a></p>
  </body>
```
Examine the policy we just applied
Take a little time to examine the policies we just applied and ensure you understand them.
Notice that the egress rule in the customer-policy is excessively liberal, allowing outbound connection to any destination. While this may be required for some workloads, in this case it is a mistake, perhaps reflecting a lazy developer. We will look at how this can be locked down further by the cluster operator or security admin in the next module.

We'll create a Calico GlobalNetworkPolicy to restrict egress to the Internet to only pods that have a ServiceAccount that is labeled "internet-egress = allowed".
Examine the network policy.
Examine the policy below. While Kubernetes network policies only have Allow rules, Calico network policies also support Deny rules. As this policy has Deny rules in it, it is important that we set its precedence higher than the lazy developer's Allow rules in their Kubernetes policy. To do this we specify order value of 600 in this policy, which gives this higher precedence than Kubernetes Network Policy (which does not have the concept of setting policy precedence, and is assigned a fixed order value of 1000 by Calico - i.e, policy order 600 gets precedence over policy order 1000).
Apply the network policy
```
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: egress-lockdown
spec:
  order: 600
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
  serviceAccountSelector: internet-egress not in {"allowed"}
  types:
  - Egress
  egress:
    - action: Deny
      destination:
        notNets:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16
          - 198.18.0.0/15
EOF
```

わかりずらいために日本語で記載
これはsa = internet egress allowed じゃないpodに対してegress traffic を制限する
制限する内容は以下のリストにないものへのegress trafffic を制限する
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
198.18.0.0/15
つまり、8.8.8.8などへアクセスをできないようにしている.

The networks referenced in the manifest above are defined by RFC5735, which is a superset of RFC1918.

Now let's try to access the internet again from the customer pod.
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```
```
ping -c 3 8.8.8.8
```
```
curl --connect-timeout 3 -I www.google.com
```
These commands should fail - pods are now restricted from accessing the internet, even if a developer included lazy egress rules to their Kubernetes network policies. You may need to terminate the command with CTRL-C. Then don't forget to exit from the pod to get back to your host terminal.
```
exit
```

Now imagine there was a legitimate reason to allow connections from the customer pod to the internet. As we used a Service Account label selector in our egress policy rules, we can enable this by adding the appropriate label to the pod's Service Account.
Add "internet-egress=allowed" to the Service Account
```
kubectl label serviceaccount -n yaobank customer internet-egress=allowed
```
Verify the pod can now access the internet
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- /bin/bash
```
```
ping -c 3 8.8.8.8
```
```
curl --connect-timeout 3 -I www.google.com
```
```
exit
```
Now you should find that the customer pod is allowed Internet Egress, but other pods (like Summary and Database) are not.

## Introduction to Protecting Hosts

Thus far, we've created policies that protect pods in Kubernetes. However, Calico Policy can also be used to protect the host interfaces in any standalone Linux node (such as a baremetal node, cloud instance or virtual machine) outside the cluster. Furthermore, it can also be used to protect the Kubernetes nodes themselves, including advanced use cases such as restricting access to NodePort services from outside the cluster.
Let's explore these more advanced scenarios, and how Calico policy can be used to protect these.
Observe that Kubernetes nodes are not yet secured
Run the command below from the standalone Host (host1) to see if we have access to FTP on the control node.
```
nc -w 3 198.19.0.1 21
```
This should succeed - i.e. you were able access the FTP server, which is not a good security posture.  (Note that we used FTP in this example as an easy illustration. Your nodes probably aren’t running an FTP daemon, but will likely be open to other attack vectors.)

### Create Network Policy for Nodes

Let's create some policies for the kubernetes nodes. Host endpoints are non-namespaced. So in order to secure host endpoints we'll need to use Calico global network policies. In a similar fashion to how we created the default-app-policy for pods in the previous module which allowed DNS but default denied all other traffic, we’ll create a default-node-policy that allows processes running in the host network namespace to connect to each other, but results in default-deny behavior for any other node connections.
```
cat <<EOF| calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-node-policy
spec:
  selector: has(kubernetes.io/hostname)
  ingress:
  - action: Allow
    protocol: TCP
    source:
      nets:
      - 127.0.0.1/32
  - action: Allow
    protocol: UDP
    source:
      nets:
      - 127.0.0.1/32
EOF
```

Create Host Endpoints.
Let’s now create the Host Endpoints, allowing Calico to start policy enforcement on node interfaces. 
First, verify there no existing Host Endpoints:
```
calicoctl get heps
```
Example output:
NAME   NODE

Now let’s configure Calico to automatically create Host Endpoints for Kubernetes nodes:
```
calicoctl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```
Within a few moments you should see Host Endpoints for each of the Kubernetes nodes:
```
calicoctl get heps
```
Example output:
NAME               NODE
node2-auto-hep     node2
control-auto-hep   control
node1-auto-hep     node1

Try to access FTP again
Run the nc again from the standalone host (host1):
```
nc -w 3 198.19.0.1 21
```
This time the netcat should fail, and timeout after 3 seconds.

Terrific, we've locked down the ftp daemon to only be accessible from the relevant places.
What about control plane traffic?
You might be wondering why the above network policy does not break the Kubernetes and Calico control planes. After all, they need to make non-local connections in order to function correctly. 
The answer is that Calico has a configurable list of “failsafe” ports which take precedence over any policy. These failsafe ports ensure the connections required for the host networked Kubernetes and Calico control planes processes to function are always allowed (assuming your failsafe ports are correctly configured). This means you don’t have to worry about defining policy rules for these. The default failsafe ports also allow SSH traffic so you can always log into your nodes.
If it wasn’t for these failsafe rules then the above policy would actually stop the Calico and Kubernetes control planes from working, and to fix the situation you would need to fix the network policy and then reboot each node so it can regain access to the control plane. So we always recommend ensuring you have the correct failsafe ports configured before you start applying network policies to host endpoints!

The doNotTrack and preDNAT and applyOnForward fields are meaningful only when applying policy to a host endpoint.

See Policy for hosts for how doNotTrack and preDNAT and applyOnForward can be useful for host endpoints.

### Restrict access to kubernetes nodeports
Lock down node port access
Kube-proxy load balances incoming connections to node ports to the pods backing the corresponding service. This process involves using DNAT (Destination Network Address Translation) to map the connection to the node port to a pod IP address and port. (We’ll dig deeper into Kubernetes services and how NAT works next week’s modules.)
Calico GlobalNetworkPolicy allows you to write policy that is enforced before this translation takes place. i.e. Policy that sees the original node port as the destination, not the backing pod that is being load balanced to as the destination. 
```
cat <<EOF | calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: nodeport-policy
spec:
  order: 100
  selector: has(kubernetes.io/hostname)
  applyOnForward: true
  preDNAT: true
  ingress:
  - action: Deny
    protocol: TCP
    destination:
      ports: ["30000:32767"]
  - action: Deny
    protocol: UDP
    destination:
      ports: ["30000:32767"]
EOF
```
Note: The port range found above (30000-32767) is the default service port range.
Verify you cannot access yaobank frontend
```
curl --connect-timeout 3 198.19.0.1:30180
```
This should fail.
Selectively allow access to customer front end
Let’s update our policy to allow only host1 to access the customer node port:
```
cat <<EOF | calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: nodeport-policy
spec:
  order: 100
  selector: has(kubernetes.io/hostname)
  applyOnForward: true
  preDNAT: true
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      ports: [30180]
    source:
      nets:
      - 198.19.15.254/32
  - action: Deny
    protocol: TCP
    destination:
      ports: ["30000:32767"]
  - action: Deny
    protocol: UDP
    destination:
      ports: ["30000:32767"]
EOF
```
Verify access
Now attempt to access the front end from host1:
```
curl 198.19.0.1:30180
```
This should succeed from host1, but will fail from any other location.
We hope this module illustrated the power of Calico for enabling advanced network security across your Kubernetes cluster, including hostports and host-networked pods, and even for services and nodePorts!

Key note.
Calico network policies can be used to enforce security within an Istio service mesh.

### Introduction to Pod Connectivity 
![Introduction to Pod Connectivity](https://user-images.githubusercontent.com/66551005/188762442-bc4e87c9-0780-4afc-be7d-4d8d494a3c18.png)
fundamental of pod connectivity
![Introduction to Pod Connectivity](https://user-images.githubusercontent.com/66551005/188762423-bedec0e2-26c1-4d35-89d4-ff8bdaa2a3cd.png)
fundamental of pod connectivity (if the nodes are same subnet)

We'll start by examining what the network looks like from the pod's point of view. Each pod gets its own Linux network namespace, which you can think of as giving it an isolated copy of the Linux networking stack.

Before we start, from host1, get the details the customer pod:
```
kubectl get pods -n yaobank -l app=customer -o wide
```
<img width="1451" alt="Screen Shot 2022-09-05 at 14 23 36" src="https://user-images.githubusercontent.com/66551005/188762412-73ce5f9d-09a3-4e7c-8aaa-ed705442ef40.png">


Note the pod’s IP address. 
Exec into the pod.
Let's exec into the pod so we can see its local view of the network (the IP addresses, network interfaces, and the routing table within the pod).
```
CUSTOMER_POD=$(kubectl get pods -n yaobank -l app=customer -o name)
```
```
kubectl exec -ti -n yaobank $CUSTOMER_POD -- /bin/bash
```
Execing into the pod means our command line is now in the context of the pod’s network namespace.

Interfaces
To begin, let's take a look at the interfaces within the pod by using ip addr.
```
ip addr
```
Example output:
<img width="1451" alt="Screen Shot 2022-09-05 at 14 28 06" src="https://user-images.githubusercontent.com/66551005/188762391-e296e54f-48ec-4dad-b9da-f5f1bb252394.png">

The key things to note in this output are:
1. There is a lo loopback interface with an IP address of 127.0.0.1. This is the standard loopback interface that every network namespace has by default. You can think of it as localhost for the pod itself.
2. There is an eth0 interface which has the pods actual IP address, 198.19.22.132. Notice this matches the IP address that kubectl get pods returned earlier.
Next let's look more closely at the interfaces using ip link. We will use the  -c option, which colors the output to make it easier to read.
```
ip -c link show up
```
Example output:
<img width="1451" alt="Screen Shot 2022-09-05 at 14 30 44" src="https://user-images.githubusercontent.com/66551005/188762383-edb15898-ee0f-4f63-ac20-1834500544cf.png">

The key things to note are:
1. eth0 is a link to the host network namespace (indicated by link-netnsid 0). This is the pod's side of the virtual ethernet pair (veth pair) that connects the pod to the node’s host network namespace.
2. The @if9 at the end of the interface name (on eth0) is the interface number for the other end of the veth pair, which is located within the host's network namespace itself.  In this example, interface number 9. Remember this number for later. You might want to write it down, because we will need to know this number when we take a look at the other end of the veth pair shortly.

Routing Table
Finally, let's look at the routes the pod sees.
```
ip route
```
Example output:
<img width="1451" alt="Screen Shot 2022-09-05 at 14 36 00" src="https://user-images.githubusercontent.com/66551005/188762365-50e4c205-eb70-42c2-a1e7-3ab2441d6553.png">

This shows that the pod's default route is out over the eth0 interface. i.e. Anytime it wants to send traffic to anywhere other than itself, it will send the traffic over eth0. (Note that the next hop address of 169.254.1.1 is a dummy address used by Calico. Every Calico networked pod sees this as its next hop.)

next hop address is that an IP address entry in a router's routing table, which specifies the next closest/most optimal router in its routing path

Exit from the customer pod
We've finished our tour of the pod's view of the network, so we'll exit out of the exec to return to host1.
```
exit
```

### How hosts see connections to Pods 
From the host1 instance we can use kubectl to find which node is running the customer pod. We can do this by running the following command:
```
kubectl get pods -n yaobank -l app=customer -o wide
```
Example output
<img width="1451" alt="Screen Shot 2022-09-05 at 15 18 29" src="https://user-images.githubusercontent.com/66551005/188762350-e5a9e343-d114-4634-a989-5efa4e475cfb.png">

SSH into whichever node is hosting the customer pod.
```
ssh node1
```
After we reach the node prompt, we will invoke the same command we used to view the active  network interfaces inside the pod.
```
ip -c link show up
```
Example output
<img width="1451" alt="Screen Shot 2022-09-05 at 15 20 23" src="https://user-images.githubusercontent.com/66551005/188762332-006ab595-5d58-4ed3-bcb8-5317e07a10aa.png">

Look for the interface number that we noted when looking at the interfaces inside the pod. In our example it was interface number 9.  Looking at interface 9 in the above output, we see caliea2aa288365 which links to @if3 in network namespace ID 3 (the customer pod's network namespace). You may recall that interface 9 in the pod's network namespace was eth0, so this looks exactly as expected for the veth pair that connects the customer pod to the host network namespace. The interface numbers in your environment may be different but you should be able to follow the same chain of reasoning.

You can also see the host end of the veth pairs to other pods running on this node, all beginning with cali.

We will look at how the node routes traffic to and from these calico interfaces to the rest of the network next.

Now we can leave node1 and return to host1 by using the exit command.
```
exit
```
First let's remind ourselves of the customer pod's IP address:
```
kubectl get pods -n yaobank -l app=customer -o wide
```
Now let's enter the node hosting the customer pod again.
```
ssh node1
```
Now let's look at the routes on the node.
```
ip route
```
Example output:
<img width="1451" alt="Screen Shot 2022-09-05 at 15 31 32" src="https://user-images.githubusercontent.com/66551005/188762313-7e785d23-4c1d-4834-99af-8dd733c37d60.png">

In this example output, we can see the route to the customer pod's IP (198.19.22.131) is via the caliabe56f96832 interface, the host end of the veth pair for the customer pod. You can see similar routes for each of the IPs of the other pods hosted on this node. It's these routes that tell Linux where to send traffic that is destined to a local pod on the node.

We can also see several routes labeled proto bird. These are routes to pods on other nodes that Calico has learned over BGP.
To understand these better, consider this route in the example output above 198.19.21.64/26 via 198.19.0.3 dev eth0 proto bird . It indicates pods with IP addresses falling within the 198.19.21.64/26 CIDR can be reached 198.19.0.3 (which is node2) through the eth0 network interface (the host's main interface to the rest of the network). You should see similar routes in your output for each node.
Calico uses route aggregation to reduce the number of routes when possible. (e.g. /26 in this example). The /26 corresponds to the default block size that Calico IPAM (IP Address Management) allocates on demand as nodes need pod IP addresses. (If desired, the block size can be configured in Calico IPAM settings.)
You can also see the blackhole 198.19.22.128/26 proto bird route. The 198.19.22.128/26 corresponds to the block of IPs that Calico IPAM allocated on demand for this node. This is the block from which each of the local pods got their IP addresses. The blackhole route tells Linux that if it can't find a more specific route for an individual IP in that block then it should discard the packet (rather than sending it out the default route to the network). You will only see traffic that hits this rule if something is trying to send traffic to a pod IP that doesn't exist, for example sending traffic to a recently deleted pod.
If Calico IPAM runs out of blocks to allocate to nodes, then it will use unused IPs from other nodes' blocks. These will be announced over BGP as more specific routes, so traffic to pods will always find its way to the right host.
Exit from the node
We've finished our tour of the Customer pod's host's view of the network. Remember to exit out of the exec to return to host1.
```
exit
```
### Encrypting data in Transit 
![Introduction to Encryption](https://user-images.githubusercontent.com/66551005/188762287-3ad2a573-6f6a-4e48-a46b-3c41cd7d39aa.png)

Calico makes it easy to encrypt on the wire in-cluster pod traffic in a Calico cluster using WireGuard. WireGuard utilizes state-of-the-art cryptography and aims to be faster, simpler, leaner, than alternative encryption techniques such as IPsec.
Calico handles all the configuration of WireGuard for you to provide full mesh encryption across all the nodes in your cluster.  WireGuard is included in the latest Linux kernel versions by default, and if running older Linux versions you can easily load it as a kernel module. (Note that if you have some nodes that don’t have WireGuard support, then traffic to/from those specific nodes will be unencrypted.)
While WireGuard performs well, there is still an overhead associated with encryption. As-such, at the time of this writing: this is an optional feature that is not enabled by default.
Let’s start by enabling encryption:
```
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```
It’s as simple as that! Within a few moments WireGuard encryption will be in place on all the nodes in the cluster.

Inspecting WireGuard status.
Every node that is using WireGuard encryption generates its own public key. You check the node status using calicoctl. If WireGuard is active on the node you will see the public key it is using in the status section. 
```
calicoctl get node node1 -o yaml
```
Example output: 
<img width="1722" alt="Screen Shot 2022-09-05 at 15 44 03" src="https://user-images.githubusercontent.com/66551005/188762268-4c57b7c3-1ce9-4046-8f63-24b24d1e50c7.png">

Let’s ssh to a node directly and inspect the interfaces present.
```
ssh node1
```
Once we’ve entered the node, we can inspect the WireGuard interface:
```
ip addr | grep wireguard
```
Example output 
<img width="1722" alt="Screen Shot 2022-09-05 at 15 45 01" src="https://user-images.githubusercontent.com/66551005/188762246-65eb4131-cbb9-4db3-b384-0d0e9b29efba.png">

Note that the WireGuard interface needs an IP address. Calico automatically allocates this from the default IP Pool.
Finally let’s exit the node returning to host1.
```
exit
```
Switching off encryption is as simple as switching it on. Let’s try that now.
```
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":false}}'
```
Within a few moments encryption will be disabled. This can be validated by looking at the node once again and seeing that the wireguard public key has been removed from the specification. 
```
calicoctl get node node1 -o yaml
```
Now that we’ve explored how easy it is to enable full cluster encryption with Calico we can proceed to the next module.

### IP pools and BGP peering

Ip pools is the calico resourses to define IP range.
One use of Calico IP Pools is to distinguish between different ranges of addresses with different routability scopes. If you are operating at very large scales then IP addresses are precious. You might want to have a range of IPs that is only routable within the cluster, and another range of IPs that is routable across the whole of your enterprise. Then you could choose which pods should get IPs from which range depending on whether workloads from outside of the cluster need to directly access the pods or not.
We'll simulate this use case in this lab by creating a second IP Pool to represent the externally routable pool, and we’ll use host1 to represent a router that is “outside of the cluster”.   (And we've already configured the lab infrastructure to not allow routing of the existing IP Pool outside of the cluster.)
Create externally routable IP Pool
We're going to create a new pool for 198.19.24.0/21 that we want to be externally routable.
```
cat <<EOF | calicoctl apply -f - 
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: external-pool
spec:
  cidr: 198.19.24.0/21
  blockSize: 29
  ipipMode: Never
  natOutgoing: true
  nodeSelector: "!all()"
EOF
```
Let’s check the new IP pools we now have:
```
calicoctl get ippools 
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-05 at 16 35 45" src="https://user-images.githubusercontent.com/66551005/188762221-05df297c-f1b8-435e-90cc-d911b19da3c5.png">

We now have:
1. 198.19.16.0/20 - Cluster Pod CIDR
2. 198.19.16.0/21 - Default IP Pool CIDR
3. 198.19.24.0/21 - External Pool CIDR
4. 198.19.32.0/20 - Service CIDR

Examine BGP peering status.
Some calicoctl commands need to be run locally on the corresponding node. The “calicoctl node status” command is one such command.

Switch to node1:
```
ssh node1
```
Check the status of Calico on the node:
```
sudo calicoctl node status
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-05 at 16 37 38" src="https://user-images.githubusercontent.com/66551005/188762211-ccbf2aad-c4df-42cc-b953-71d80e174c51.png">

This shows that currently this node is only peering with the other nodes in the cluster and is not peering to any networks outside of the cluster.
As a reminder of what we covered in the earlier videos, Calico adds routes on each node to the local pods on that node. Note that BGP is not involved in programming the routes to the local pods on the node. Each node only uses BGP to share these local routes with the rest of the network, and to learn routes from the rest of the network which it then adds to the node.
Exit back to host1:
```
exit
```

Add a BGP Peer.
In this lab we will simulate peering to a network outside of the cluster by peering to host1. (We've set up host1 to act as if it were a router, and it is ready to accept new BGP peering requests.)
Add the new BGP Peer:
```
cat <<EOF | calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-host1
spec:
  peerIP: 198.19.15.254
  asNumber: 64512
EOF
```
Examine the new BGP peering status
Switch to node1 again:
```
ssh node1
```
Check the status of Calico on the node:
```
sudo calicoctl node status
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-06 at 10 27 40" src="https://user-images.githubusercontent.com/66551005/188762189-f7db014d-44b4-4498-9e8f-259c51693f0b.png">

The output shows that Calico is now peered with host1 (198.19.15.254). This means Calico can share routes to and learn routes from host1. (Remember we are using host1 to represent a router in this lab.)
In a real-world on-prem deployment you would typically configure Calico nodes within a rack to peer with the ToRs (Top of Rack) routers, and the ToRs are then connected to the rest of the enterprise or data center network. In this way pods, if desired, can be addressed from anywhere on your network. You could even go as far as giving some pods public IP addresses and have them addressable from the internet if you wanted to.
We're done with adding the peers, so exit from node1 to return back to host1:
```
exit
```

Calico supports annotations on both namespaces and pods that can be used to control which IP Pool (or even which IP address) a pod will receive when it is created. In this example we're going to create a namespace to host out an externally routable network.
Create the namespace
Notice the annotation that will determine which IP Pool pods in the namespace will use.
Apply the namespace:
```
cat <<EOF| kubectl apply -f - 
---
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    cni.projectcalico.org/ipv4pools: '["external-pool"]'
  name: external-ns
EOF
```
Now deploy a NGINX example pod in the external-ns namespace, along with a simple network policy that allows ingress on port 80.
```
kubectl apply -f https://raw.githubusercontent.com/tigera/ccol1/main/nginx.yaml
```
Let's see what IP address was assigned:
```
kubectl get pods -n external-ns -o wide
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-06 at 10 42 35" src="https://user-images.githubusercontent.com/66551005/188762163-d9ece457-54d7-4670-8e98-69f5123f182d.png">

The output shows that the nginx pod has an IP address from the externally routable IP Pool.
Try to connect to it from host1:
```
curl 198.19.28.208
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-06 at 10 43 02" src="https://user-images.githubusercontent.com/66551005/188762142-5c04cc10-501a-4a68-b6e5-f357d69ba131.png">

This confirms that the NGINX pod is directly routable on the broader network. (In this simplified lab this means host1, but in a real environment this could be across the whole of the enterprise network if desired.)

Let’s also take a quick look at the IP allocation stats from Calico-IPAM, by running the following command:
```
calicoctl ipam show
```
Example output:
<img width="1722" alt="Screen Shot 2022-09-06 at 10 44 00" src="https://user-images.githubusercontent.com/66551005/188762123-241b4b47-8dae-463e-9b41-64394b99b54d.png">

It can be a good idea to periodically check IP allocations statistics to check you have sized your IP pools appropriately, for example if you aren’t confident about the number of pods and whether your original sizing of the pools.


Key notes:
overlay networks
1. encapsulate pod to pod packets inside node to node packets
2. can be implemented using IPIP
3. can be implemented using VXLAN
4. can be implemented using WireGuard with the added benefit of encryption

wireGuard 
1. can be thought of as an overlay network with the added benefit of encryption
2. uses status of the art encryption
3. can be used by Calico to secure all pod to pod traffic over the underlying network

Calico IP pools 
1. Define range of IP addresses that can be used for Calico IPAM
2. Define IP range specific network behaviors such as overlay modes or NAT outgoing
3. Can be considered to only be used by specific nodes, namespace, or pods
4. Define the block sizes to be used in BGP route aggregation

BGP is 
1. a standard based routing protocol supported by most routers
2. used to build the internet 
3. can be used between calico nodes to share routes 
4. can be used to share routes between calico and the underlying network 
5. can be used to share service IPs with the underlying network 
6. often used in on-prem or private cloud newworks 

Calico networking is 
1. connects pods to the host using veth pairs 
2. configures the host to act as a virtual router 
3. programs local routes on each host for each of the pods on the host 
4. can use BGP if wanted 
5. can run as an overlay if wanted 

kubernetes services in details 
サービスとは？
IPアドレスが変動するポッドに対してクライアントが一意の宛先でアクセスするために作成するオブジェクト
ClusterIPは各ポッド間の通信で利用するものですがNodePortはKubernetesクラスタ外からもアクセスが可能
クラスター外のロードバランサーに外部と通信可能な仮想 IP アドレスを払い出す

Let's take a look at the services and pods in the yaobank kubernetes namespace.
List the services:
```
kubectl get svc -n yaobank
```
Example output:
<img width="895" alt="CLUSTER-IP" src="https://user-images.githubusercontent.com/66551005/188762091-382f8b3f-745a-46c5-8d8a-fd8583ab4c09.png">

We should have three services deployed. One NodePort service (used for the customer front end when accessed from outside the cluster) and two ClusterIP services (used for the summary and database when accessed inside the cluster).
Now let’s look at the endpoints for each of the services:
```
kubectl get endpoints -n yaobank
```
Example output:
<img width="895" alt="database 98 19 21 672379" src="https://user-images.githubusercontent.com/66551005/188762083-c0b3ef0f-f060-48e5-aefe-ef39a87fd731.png">

Now list the pods:
```
kubectl get pods -n yaobank -o wide
```
Example output:
<img width="1608" alt="Screen Shot 2022-09-06 at 12 08 53" src="https://user-images.githubusercontent.com/66551005/188762069-54f2b8c9-3c35-44df-8232-d0d16517195c.png">

You can see that the IP addresses listed as the service endpoints in the previous step are the IP addresses of the pods backing each service, as expected. Each service is backed by one or more pods spread across the worker nodes in our cluster.

In this cluster we are currently running kube-proxy in its default, and most commonly used, iptables mode (not IPVS mode).
To explore the iptables rules kube-proxy programs into the kernel to implement Cluster IP based services, let's look at the Database service. The Summary pods use this service to connect to the Database pods. Kube-proxy uses DNAT to map the Cluster IP to the chosen backing pod.

Get service endpoints
Find the service endpoints for summary ClusterIP service:
```
kubectl get endpoints -n yaobank summary
```
Example output:
<img width="1608" alt="Screen Shot 2022-09-06 at 12 15 19" src="https://user-images.githubusercontent.com/66551005/188762054-d1bbd706-6a62-4c34-888b-ec8a81ad34fb.png">

The summary service has two endpoints (198.19.21.1 on port 80, and 198.19.22.131 on port 80, in this example output). Starting from the KUBE-SERVICES iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.
SSH into a node
Let’s SSH into node1 so we can explore the iptables rules that kube-proxy has set up on that node. (It will have set up similar rules on every other node on the cluster.)
```
ssh node1
```
Examine the KUBE-SERVICE chain
To make it easier to manage large numbers of iptables rules, groups of iptables rules can be grouped together into iptables chains. Kube-proxy puts its top level rules into a KUBE-SERVICES chain.
Let’s take a look at that chain now:
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 12 23 07" src="https://user-images.githubusercontent.com/66551005/188762038-a7a23c36-3a37-4baa-a684-03c13126371d.png">

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
1. target - which chain iptables will jump to if the rule matches
2. prot - the protocol match criteria
3. source, and destination - the source and destination IP address match criteria
4. the comments that kube-proxy includes
5. the additional match criteria at the end of each rule - e.g dpt:80 that specifies the destination port match.

You can see this chain includes rules to jump to service specific chains, one for each service.
KUBE-SERVICES -> KUBE-SVC-XXXXXXXXXXXXXXXX
Let's look more closely at the rules for the summary service.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 12 23 58" src="https://user-images.githubusercontent.com/66551005/188762016-74ee74d6-b5e4-4cc9-9b49-30f6e0eea36e.png">

The second rule directs traffic destined for the summary service clusterIP (198.19.44.145 in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).
KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
kube-proxy in iptables mode uses a randomized equal cost selection algorithm to load balance traffic between pods. We currently have two summary pods, so it should have rules in place that load balance equally across both pods.
Let's examine how this load balancing works using the chain name returned from our previous command. (Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 12 30 25" src="https://user-images.githubusercontent.com/66551005/188762007-d4447fac-7bc0-4dd6-9052-ff11af2c1e50.png">

Notice that kube-proxy is using the iptables statistic module to set the probability for a packet to be randomly matched.
The first rule directs traffic destined for the summary service to a chain that delivers packets to the first service endpoint (KUBE-SEP-XSAPI3SEG2KJVDTP) with a probability of 0.50000000000. The second rule unconditionally directs to the second service endpoint chain (KUBE-SEP-KGQRHMYN3EBQMPXF). The result is that traffic is load balanced across the service endpoints equally (on average).
If there were 3 service endpoints then the first chain matches would be probability 0.33333333, the second probability 0.5, and the last unconditional. The result of this is that each service endpoint receives a third of the traffic (on average).
And so on for any number of services!
KUBE-SEP-XXXXXXXXXXXXXXXX -> summary pod
Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-KGQRHMYN3EBQMPXF
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 12 37 17" src="https://user-images.githubusercontent.com/66551005/188761995-e818a101-14be-4cbb-91bc-04813bd78c6e.png">

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (198.19.22.130 in this example). After this, standard Linux routing can handle forwarding the packet like it would any other packet.
Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to summary pods exposed as a service of type ClusterIP.
In summary, for a packet being sent to a clusterIP:
The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).
Finally, let's return to host1 to take a look at NodePorts in the next section by using the exit command.
```
exit
```
Let's explore the iptables rules that implement the customer service with a node port.
To explore the iptables rules kube-proxy programs into the kernel to implement Node Port based services, let's look at the Customer service. External clients use this service to connect to the Customer pods.  Kube-proxy uses NAT to map the Node Port to the chosen backing pod, and the source IP to the node IP of the ingress node, so that it can reverse the NAT for return packets.  (If it didn't change the source IP then return packets would go directly back to the client, without the node that did the NAT having a chance to reverse the NAT, and as a result the client would not recognize the packets as being part of the connection it made to the Node Port).

Get service endpoints
Find the service endpoints for customer NodePort service.
```
kubectl get endpoints -n yaobank customer
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 13 15 22" src="https://user-images.githubusercontent.com/66551005/188761984-16ea3d91-db74-4876-ae50-4baeb8b31633.png">

The customer service has one endpoint (198.19.22.131 on port 80 in this example output). Starting from the KUBE-SERVICES iptables chain, we will traverse each chain until you get to the rule directing traffic to this endpoint IP address.
KUBE-SERVICES -> KUBE-NODEPORTS
Lets re-enter the node1 instance to look at iptables once again with this new information.
```
ssh node1
```
The KUBE-SERVICE chain handles the matching for service types ClusterIP and LoadBalancer. At the end of KUBE-SERVICE chain, another custom chain KUBE-NODEPORTS will handle traffic for service type NodePort.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 13 17 02" src="https://user-images.githubusercontent.com/66551005/188761972-44fc3c20-5814-4e5c-9931-f2d6dedfac30.png">

“match dst-type LOCAL” matches any packet with a local host IP as the destination. I.e. any address that is assigned to one of the host's interfaces.
KUBE-NODEPORTS -> KUBE-SVC-XXXXXXXXXXXXXXXX
```
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 13 17 52" src="https://user-images.githubusercontent.com/66551005/188761959-b994b7b4-1e60-455d-9dd8-7788d4244bc5.png">

The second rule directs traffic destined for the customer service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT). tcp dpt:30180 matches any packet with the destination port of tcp 30180 (the node port of the customer service).
KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 13 54 27" src="https://user-images.githubusercontent.com/66551005/188761948-8edb5053-ffe6-4044-a664-07075c1e996f.png">

As we only have a single backing pod for the customer service, there is no load balancing to do, so there is a single rule that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-3ZSIELULTQZPQFWS).
KUBE-SEP-XXXXXXXXXXXXXXXX -> customer endpoint
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-3ZSIELULTQZPQFWS
```
Example output:
<img width="1657" alt="Screen Shot 2022-09-06 at 13 56 59" src="https://user-images.githubusercontent.com/66551005/188761933-1b1fce8e-0363-4a43-ab62-bd95b3e97c73.png">

This rule delivers the packet to the customer service endpoint.
The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (198.19.22.132 in this example). After this, standard Linux routing can handle forwarding the packet like it would any other packet.
Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to customer pods exposed as a service of type NodePort.
In summary, for a packet being sent to a NodePort:
The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
1. The KUBE-NODEPORTS chain matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
2. The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
3. The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).
Exit back to host1
That’s it! We're done with this module. Exit the ssh session to node1 and return to host1.
```
exit
```
### Calico native service handing
About Calico eBPF
Calico's eBPF dataplane is an alternative to the default standard Linux dataplane (which is iptables based). The eBPF dataplane has a number of advantages:
1. It scales to higher throughput.
2. It uses less CPU per GBit.
3. It has native support for Kubernetes services (without needing kube-proxy) that:
   - Reduces first packet latency for packets to services.
   - Preserves external client source IP addresses all the way to the pod.
   - Supports DSR (Direct Server Return) for more efficient service routing.
   - Uses less CPU than kube-proxy to keep the dataplane in sync.
The eBPF dataplane also has some limitations, which are described in the Enable the eBPF dataplane guide in the Calico documentation.

To enable Calico eBPF we need to:
1. Configure Calico so it knows how to connect directly to the API server (rather than relying on kube-proxy to help it connect)
2. Disable kube-proxy
3. Configure Calico to switch to the eBPF dataplane
Configure Calico to connect directly to the API server
In eBPF mode, Calico replaces kube-proxy. This means that Calico needs to be able to connect directly to the API server (just as kube-proxy would normally do). Calico supports a ConfigMap to configure these direct connections for all of its components.
Note: It is important the ConfigMap points at a stable address for the API server(s) in your cluster. If you have a HA cluster, the ConfigMap should point at the load balancer in front of your API servers so that Calico will be able to connect even if one control plane node goes down. In clusters that use DNS load balancing to reach the API server (such as kops and EKS clusters) you should configure Calico to talk to the corresponding domain name.
In our case, we have a single control node hosting the Kubernetes API service. So we will just configure the control node IP address directly:
```
cat <<EOF | kubectl apply -f -
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "198.19.0.1"
  KUBERNETES_SERVICE_PORT: "6443"
EOF
```
ConfigMaps can take up to 60s to propagate; wait for 60s and then restart the operator, which itself also depends on this config:
```
kubectl delete pod -n tigera-operator -l k8s-app=tigera-operator
```
If you watch the Calico pods, you should see them get recreated with the new configuration:
```
watch kubectl get pods -n calico-system
```
You can use Ctrl+C to exit from the watch command once all of the Calico pods have come up.
Disable kube-proxy
Calico’s eBPF native service handling replaces kube-proxy. You can free up resources from your cluster by disabling and no longer running kube-proxy. When Calico is switched into eBPF mode it will try to clean up kube-proxy's iptables rules if they are present. 
Kube-proxy normally runs as a daemonset. So an easy way to stop and remove kube-proxy from every node is to add a nodeSelector to that daemonset which excludes all nodes.
In some environments, kube-proxy is installed/run via a different mechanism. For example, a k3s cluster typically doesn’t have a kube-proxy daemonset, and instead is controlled by install time k3s configuration.  In this case, you would still want to get rid of kube-proxy, but if you just wanted to try out Calico eBPF quickly on such a cluster, you can tell Calico to not try to tidy up kube-proxy’s iptables rules, and instead allow them to co-exist.  (Calico eBPF will still bypass the iptable rules, so they have no effect on the traffic.)
Let’s do that now in this cluster by running this command on host1:
```
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfKubeProxyIptablesCleanupEnabled": false}}'
```
Switch on eBPF mode
You're now ready to switch on eBPF mode. To do so, on host1, use calicoctl to enable the eBPF mode flag:
```
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfEnabled": true}}'
```
Since enabling eBPF mode can disrupt existing connections, restart YAO Bank's customer and summary pods:
```
kubectl delete pod -n yaobank -l app=customer
kubectl delete pod -n yaobank -l app=summary
```
You're now ready to proceed to the next module to verify connectivity.

Now that we've switched to the Calico eBPF data plane, Calico native service handling handles the service without needing kube-proxy. As it is handling the packets with eBPF, rather than the standard Linux networking pipeline, it is able to special case this traffic in a way that allows it to preserve the source IP address, including special handling of return traffic so it still is returned from the original ingress node.

So we can see the effect of source IP preservation, tail the logs of YAO Bank's customer pod again in your second shell window:
```
kubectl logs -n yaobank -l app=customer --follow
```
While the above command is running, access YAO Bank from your other host1 shell, using the node port via the control node:
```
curl 198.19.0.1:30180
```
You should see these logs from the customer pod appear:
<img width="1095" alt="Debug mode off" src="https://user-images.githubusercontent.com/66551005/188761909-379de954-c076-4fa8-bd92-a3973963dc63.png">

This time the source IP that the pod sees is 198.19.15.254, which is host1, which was the real source of the request, showing that the source IP has been preserved end-to-end.
This is great for making logs and troubleshooting easier to understand, and means you can now write network policies for pods that restrict access to specific external clients if desired. 

Calico’s eBPF dataplane also supports DSR (Direct Server Return). DSR allows the node hosting a service backing pod to send return traffic directly to the external client rather than taking the extra hop back via the ingress node (the control node in our example). 

DSR requires a network fabric with suitably relaxed RPF (reverse path filtering) enforcement. In particular the network must accept packets from nodes that have a source IP of another node.  In addition, any load balancing or NAT that is done outside the cluster must be able to handle the DSR response packets from all nodes.
Snoop traffic without DSR
To show the effect of this, let’s snoop the traffic on the control node.
SSH into the control node:
```
ssh control
```
Snoop the traffic associated with the node port:
```
sudo tcpdump -nvi any 'tcp port 30180'
```
Example output:
<img width="1095" alt="Screen Shot 2022-09-06 at 15 01 49" src="https://user-images.githubusercontent.com/66551005/188761902-245bb6cf-f1bb-4748-b8bc-1045bd12c591.png">

While the above command is running, access YAO Bank from your other host1 shell, using the node port via the control node:
```
curl 198.19.0.1:30180
```
The traffic will get logged by the tcpdump, for example:
<img width="1714" alt="Screen Shot 2022-09-06 at 15 03 16" src="https://user-images.githubusercontent.com/66551005/188761887-20b442cf-b08f-4b39-ad97-959d90ec1620.png">

You can see there is traffic flowing via the node port in both directions. In this example:
1. Traffic from host1 to the node port: 198.19.15.254.57064 > 198.19.0.1.30180 
2. Traffic from the node port to host1: 198.19.0.1.30180 > 198.19.15.254.57064 
Leave the tcpdump command running and we’ll see what difference turning on DSR makes.
Switch on DSR
Run the following command from host1 to turn on DSR:
```
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
```
Now access YAO Bank from host1 using the node port via the control node:
```
curl 198.19.0.1:30180
```
The traffic will get logged by the tcpdump, for example:
<img width="1714" alt="Screen Shot 2022-09-06 at 15 06 18" src="https://user-images.githubusercontent.com/66551005/188761876-955cc658-869f-40e2-b134-4b1d47029c7e.png">

You should only see traffic in one direction, from host1 to the node port on the control node. The return traffic is going directly back to the client (host1) from the node hosting the customer pod backing the service (node1).
Exit back to host1
You can now ctrl-c to terminate the tcpdump and then exit from the control back to the host1 command line.
```
exit
```
### Advertising services
Typically, Kubernetes service cluster IPs are accessible only within the cluster, so external access to the service requires a dedicated load balancer or ingress controller. In cases where a service’s cluster IP is not routable, the service can be accessed using its external IP.
Just as Calico supports advertising pod IPs over BGP, it also supports advertising Kubernetes service IPs outside a cluster over BGP. This avoids the need for a dedicated load balancer. This feature also supports equal cost multi-path (ECMP) load balancing across nodes in the cluster, as well as source IP address preservation for local services when you need more control.

Advertising services over BGP allows you to directly access the service without using NodePorts or a cluster Ingress Controller.
Examine routes
Let’s start by taking a look at the state of routes on host1:
```
ip route
```
Example output:
<img width="1636" alt="Screen Shot 2022-09-06 at 15 16 25" src="https://user-images.githubusercontent.com/66551005/188761861-324e2c3b-b00d-45ce-aa61-8ab8840aae13.png">

If you completed the previous lab you'll see one route that was learned from Calico that provides access to the nginx pod that was created in the externally routable namespace (the route ending in “proto bird”, the last line in this example output). In this lab we will advertise Kubernetes services (rather than individual pods) over BGP.

Update Calico BGP configuration
The serviceClusterIPs clause below tells Calico to advertise the cluster IP range.
Apply the configuration:
```
cat <<EOF | calicoctl apply -f -
---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: "198.19.32.0/20"
EOF
```
Verify the BGPConfiguration update worked and contains the serviceClusterIPs key:
```
calicoctl get bgpconfig default -o yaml
```
Example output:
<img width="691" alt="ubuntu@hostl~$ calicoctl get bgpconfig default -o yaml" src="https://user-images.githubusercontent.com/66551005/188761848-f93b7cb2-2803-4399-8394-3b323cfb52b1.png">

Examine routes
Examine the routes again on host1:
```
ip route
```
Example output:
<img width="755" alt="192 168 64 1 dev enp0s2 proto dhcp scope link 192 168 64 6 metri 100" src="https://user-images.githubusercontent.com/66551005/188761838-9ddf5aa1-c283-4b11-b944-5c47da3dc168.png">

You should now see the cluster service cidr 198.19.32.0/20 advertised from each of the kubernetes cluster nodes. This means that traffic to any service's cluster IP address will get load balanced across all nodes in the cluster by the network using ECMP (Equal Cost Multi Path). Kube-proxy or Calico native service handling then load balances the cluster IP across the service endpoints (backing pods) in exactly the same way as if a pod had accessed a service via a cluster IP.
Verify we can access cluster IPs
Find the cluster IP for the customer service:
```
kubectl get svc -n yaobank customer
```
Example output:
<img width="755" alt="Screen Shot 2022-09-06 at 15 28 31" src="https://user-images.githubusercontent.com/66551005/188761823-99da7a92-d01e-46af-b5ef-1fc92a67d188.png">

Confirm we can access it from host1:
```
curl 198.19.35.252
```
Advertising Cluster IPs in this way provides an alternative to accessing services via Node Ports (simplifying client service discovery without clients needing to understand DNS SRV records) or external network load balancers (reducing overall equipment costs).  

If you want to advertise a service using an IP address outside of the service cluster IP range, you can configure the service to have one or more external-IPs.
Examine the existing services
Before we begin, examine the kubernetes services in the yaobank kubernetes namespace:
```
kubectl get svc -n yaobank
```
Example output: 
<img width="755" alt="yaobank" src="https://user-images.githubusercontent.com/66551005/188761805-93743600-99c8-40a5-8c0d-38cea76d6cbb.png">

Note that none of them currently have an EXTERNAL-IP.
Update BGP configuration
Update the Calico BGP configuration to advertise a service external IP CIDR range of 198.19.48.0/20:
```
calicoctl patch BGPConfig default --patch \
   '{"spec": {"serviceExternalIPs": [{"cidr": "198.19.48.0/20"}]}}'
```
Note that serviceExternalIPs is a list of CIDRs, so you could for example add individual /32 IP addresses if there were just a small number of specific IPs you wanted to advertise.
Examine routes on host1:
```
ip route
```
Example output:
<img width="755" alt="198 19 28 20829 via 198 19 0 2 dev enp8s2 proto bird" src="https://user-images.githubusercontent.com/66551005/188761795-4df2081d-8035-416d-8277-087c16025fce.png">

You should now have a route for the external ID CIDR (198.19.48.0/20) with next hops to each of our cluster nodes.

Assign the service external IP 198.19.48.10/20 to the customer service.
```
kubectl patch svc -n yaobank customer -p  '{"spec": {"externalIPs": ["198.19.48.10"]}}'
```
Examine the services again to validate everything is as expected:
```
kubectl get svc -n yaobank
```
Example output:
<img width="755" alt="CLUSTER-IP" src="https://user-images.githubusercontent.com/66551005/188761778-b9e3cf90-7956-45a8-98b7-ccb927f9d3b3.png">

You should now see the external ip (198.19.48.10) assigned to the customer service. We can now access the customer service from outside the cluster using the external ip address (198.19.48.10) we just assigned.
Verify we can access the service's external IP
Connect to the customer service from the standalone node using the service external IP 198.19.48.10:
```
curl 198.19.48.10
```
As you can see the service has been made available outside of the cluster via bgp routing and network load balancing.

## Review 
We've covered five different ways for connecting to your pods from outside the cluster during this Module.
1. Via a standard NodePort on a specific node. (This is how you connected to the YAO Bank web front end when you first deployed it.)
2. Direct to the pod IP address by using externally routable IP Pools.
3. Advertising the service cluster IP range. (And using ECMP to load balance across all nodes in the cluster.)
4. Advertising individual cluster IPs. (Services with externalTrafficPolicy: Local, using ECMP to load balance only to the nodes hosting the pods backing the service.)
5. Advertising service external-IPs. (So you can use service IP addresses outside of the cluster IP range.)


Key notes: 

kubernetes services 
1. Can be thought of as a virtual load balancer built into the pod network
2. Normally use label selectors to define which pods belong to a Service
3. Are discoverable by pods through DNS (kube-dns)
4. May include external load balancers

Cluster IP services
1. Preserve pod source IP addresses all the way to the backing pods
2. NAT the destination IP as part of load balancing to the backing pods
3. Can be discovered using DNS (kube-dns)
4. Can be advertised over BGP

Nodeport services when using kube-proxy
1. NAT the source IP as part of load balancing to the backing pods
2. NAT the destination IP as part of load balancing to the backing pods

Load balancer services typically:
1. Use external network load balancers
2. Use node ports
3. Preserve source IP for services with externalTrafficPolicy:local

Kube-proxy: 
1. Intercepts connections to services using rules it has programmed in the kernel
2. Load balances connections to services to the pods backing the service
3. Can use either iptables or IPVS rules for load balancing
4. Scales to thousands of services

Kube-proxy IPVS mode
1. Scales to thousands of services
2. Uses less CPU than iptables with thousands of services

Calico native service handling
1. Replaces kube-proxy
2. Is implemented by the Calico eBPF dataplane
3. Always preserves client source IP addresses
4. Optionally supports DSR (Direct Server Return)
5. Scales to thousands of services
6. Has lower latency and uses less CPU than kube-proxy

Calico can use BGP to
1. Advertise the cluster IP range of services
2. Advertise external IP range of services
3. Enable the underlying network to load balance services without a load balancer

***I got certified for Calico Operator: Level 1 !!!*** 
![CALICO](https://user-images.githubusercontent.com/66551005/188761737-963ffced-0e80-4b1f-bbf2-07c068d32823.png)
