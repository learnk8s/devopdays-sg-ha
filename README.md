# Build and breaking Kubernetes clusters

What is a Kubernetes cluster made of?

In this session you will experience first-hand Kubernetes' architectural design choices.

The plan is as follow:

- You will create a three nodes.
- You will bootstrap a cluster with `kubeadm` — a tool designed to create Kubernetes clusters.
- You will deploy a demo application with two replicas.
- One by one, you will take down each Node and inspect the status of the cluster.

_Will Kubernetes recover from the failures?_

There's only one way to know!

## Prerequisites

Make sure you have the following tools installed:

- Docker Desktop (or Podman Desktop)
- kubectl
- minikube

## Bootstrapping a cluster

Let's start by creating a three-node Kubernetes cluster with two worker nodes.

Instead of using a premade cluster, such as the one you can find on the major cloud providers, you will go through bootstrapping a cluster from scratch.

_But before you can create a cluster, you need nodes._

You will use minikube for that:

```bash
$ minikube start --no-kubernetes --container-runtime=containerd --driver=docker --nodes 3
😄  minikube v1.29.0
✨  Using the docker driver based on user configuration
👍  Starting minikube without Kubernetes in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
📦  Preparing containerd 1.6.15
🏄  Done! minikube is ready without Kubernetes!
```

It may take a moment to create those Ubuntu instances depending on your setup.

You can verify that the nodes are created correctly with:

```bash
$ minikube node list
minikube      192.168.105.18
minikube-m02  192.168.105.19
minikube-m03  192.168.105.20
```

It's worth noting that those nodes are not vanilla Ubuntu images.

Containerd (the container runtime) is preinstalled.

Apart from that, there's nothing else.

**It's time to install Kubernetes!**

## Bootstrapping the primary node

In this section, you will install Kubernetes on the master Node and bootstrap the control plane.

The control plane is made of the following components:

- **etcd**, a consistent and highly-available key-value store.
- **kube-apiserver**, the API you interact with when you use `kubectl`.
- **kube-scheduler**, used to schedule Pods and assign them to Nodes.
- **kube-controller-manager**, a collection of controllers used to reconcile the state of the cluster.

You can SSH into the primary node with:

```bash
$ minikube ssh --node minikube
```

If you find that terminal handling is not working well (e.g. resizing terminals don't work, command prompt behaves weird), you can try this alternative:

```bash
$ docker exec -it minikube su - docker
```

There are several tools designed to bootstrap clusters from scratch.

However, `kubeadm` is an official tool and the best supported.

You will use that to create your cluster.

To install `kubeadm` and a few more prerequisites, execute the following script in the primary node:

```bash
$ curl -s -o master.sh https://academy.learnk8s.io/master.sh
$ sudo bash master.sh auto
```

In a new terminal session, SSH into the second node with:

```bash
$ minikube ssh --node minikube-m02
```

And execute the following setup script:

```bash
$ curl -s -o worker.sh https://academy.learnk8s.io/worker.sh
$ sudo bash worker.sh auto
```

Repeat the same steps for the last node.

In a new terminal session, SSH into the third node with:

```bash
$ minikube ssh --node minikube-m03
```

And execute the following setup script:

```bash
$ curl -s -o worker.sh https://academy.learnk8s.io/worker.sh
$ sudo bash worker.sh auto
```

Those scripts:

- [Downloads `kubeadm`, `kubectl` and the `kubelet`.](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
- [Installs the shared certificates necessary to trust other entities.](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#custom-certificates)
- [Creates the Systemd unit necessary to launch the `kubelet`.](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd)
- [Creates the `kubeadm` config necessary to bootstrap the cluster.](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/)

Once completed, you can finally switch back to the terminal session for the primary node and bootstrap the cluster with:

```bash
$ sudo kubeadm init --config config.yaml
# truncated output
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control plane was initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 192.168.49.2:8443 --token nsyxx6.quc5x0djkjdr564u \
    --discovery-token-ca-cert-hash sha256:3bc332011691454867232397bf837dbb73affc96…
```

**Please make a note of the join command at the end of `kubeadm init` output. You will need it later to join the workers, and you don't have to run it now.**

The command is similar to:

```
kubeadm join <master-node-ip>:8443 --token [some token] \
  --discovery-token-ca-cert-hash [some hash]
```

The command is necessary to join other nodes in the cluster.

> **Please don't skip the previous step!** Make a note of the command and write it down! You will need it later on.

In the output of the `kubeadm init`, you can notice this part:

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

The above instructions are necessary to configure `kubectl` to talk to the control plane.

You should go ahead and follow those instructions.

Once you are done, you can verify that `kubectl` is configured correctly with:

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://<master-node-ip>:8443
CoreDNS is running at https://<master-node-ip>:8443/api/v1/namespaces/kube…
```

Let's check the pods running in the control plane with:

```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS
kube-system   coredns-787d4945fb-knjnx           0/1     ContainerCreating
kube-system   coredns-787d4945fb-vmnzn           0/1     ContainerCreating
kube-system   etcd-minikube                      1/1     Running
kube-system   kube-apiserver-minikube            1/1     Running
kube-system   kube-controller-manager-minikube   1/1     Running
kube-system   kube-proxy-wtdc4                   1/1     Running
kube-system   kube-scheduler-minikube            1/1     Running
```

_Why are the CoreDNS pods in the "ContainerCreating" state?_

Let's investigate further:

```bash
$ kubectl describe pod coredns-787d4945fb-knjnx -n kube-system
Name:                 coredns-787d4945fb-knjnx
Namespace:            kube-system
# truncated output
Events:
Type     Reason                   Message
----     ------                   -------
Warning  FailedCreatePodSandBox   Failed to create pod sandbox: ...(truncated)... network ...
```

The message suggests that the network is not ready!

_But how is that possible?_

_You are using kubectl to send commands to the control plane — it should be ready?_

The message is cryptic, but it tells you that you must still configure the network plugin.

**In Kubernetes, there is no standard or default network setup.**

Instead, you should configure your network and install the appropriate plugin.

You can choose from several network plugins, but for now, you will install Flannel — one of the simplest.

## Installing a network plugin

Kubernetes imposes the following networking requirements on the cluster:

1. All Pods can communicate with all other pods.
1. Agents on a node, such as system daemons, kubelet, etc., can communicate with all pods on that node.

Those requirements are generic and can be satisfied in several ways.

**That allows you to decide how to design and operate your cluster network.**

In this case, each node in the cluster has a fixed IP address, and you only need to assign Pod IP addresses.

[Flannel](https://github.com/flannel-io/flannel) is a network plugin that:

1. Assigns a subnet to every node.
1. Assigns IP addresses to Pods.
1. Maintains a list of Pods and Nodes in the cluster.

In other words, Flannel can route the traffic from any Pod to any Pod — just what we need.

_Let's install it in the cluster._

The `master.sh` script you executed earlier also created a `flannel.yaml` in the local directory.

```bash
$ ls -1
config.yaml
flannel.yaml
master.sh
traefik.yaml
```

You can submit it to the cluster with:

```bash
$ kubectl apply -f flannel.yaml
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

It might take a moment to download the container and create the Pods.

You can check the progress with:

```bash
$ kubectl get pods --all-namespaces
```

Once all the Pods are "Ready", the control plane Node should transition to a _"Ready"_ state too:

```bash
$ kubectl get nodes -o wide
NAME       STATUS   ROLES           VERSION
minikube   Ready    control-plane   v1.26.2
```

_Excellent!_

The control plane is successfully configured to run Kubernetes.

This time, CoreDNS should be running as well:

```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS
kube-system   coredns-787d4945fb-knjnx           1/1     Running
kube-system   coredns-787d4945fb-vmnzn           1/1     Running
kube-system   etcd-minikube                      1/1     Running
kube-system   kube-apiserver-minikube            1/1     Running
kube-system   kube-controller-manager-minikube   1/1     Running
kube-system   kube-proxy-wtdc4                   1/1     Running
kube-system   kube-scheduler-minikube            1/1     Running
```

However, there must be more than a control plane to run workloads.

_You also need worker nodes._

## Connecting worker Nodes

In the control plane node, pay attention to the running nodes:

```bash
$ watch kubectl get nodes -o wide
```

The `watch` command executes the `kubectl get nodes` command at a regular interval.

You will use this terminal session to observe nodes as they join the cluster.

In the other terminal, first, list the IP address of the second node.

```bash
$ minikube node list
minikube      192.168.105.18
minikube-m02  192.168.105.19
minikube-m03  192.168.105.20
```

Then, you should SSH into the first worker node with:

```bash
$ minikube ssh -n minikube-m02
```

Download and execute the following script to install the prerequisites:

You should now join the worker Node to the cluster with the `kubeadm join` command you saved earlier.

The command should look like this (the only thing we changed is added `sudo` in front):

```bash
$ sudo kubeadm join <master-node-ip>:8443 --token [some token] \
  --discovery-token-ca-cert-hash [some hash]
```

Execute the command and pay attention to the terminal window in the control plane Node.

> If you encounter a "Preflight Check Error", append the following flag to the `kubeadm join` command: `--ignore-preflight-errors=SystemVerification`.

**The worker node is provisioned and transitions to the "Ready" state.**

As soon as the command finishes, execute the following lines to enable kubelet to start after worker node reboot:

```bash
$ sudo systemctl enable kubelet
```

And finally, you should repeat the instructions to join the second worker node.

First, list the IP address of the third node with:

```bash
$ minikube node list
minikube      192.168.105.18
minikube-m02  192.168.105.19
minikube-m03  192.168.105.20
```

Then, SSH into the node with:

```bash
$ minikube ssh -n minikube-m03
```

Download and install the prerequisites (pay attention to the new IP address):

Join the node to the cluster with the same `kubeadm join` command you used earlier:

```bash
$ sudo kubeadm join <master-node-ip>:8443 --token [some token] \
  --discovery-token-ca-cert-hash [some hash]
```

> If you encounter a "Preflight Check Error", append the following flag to the `kubeadm join` command: `--ignore-preflight-errors=SystemVerification`.

You should observe even the second worker joining the cluster and transitioning to the "Ready" state.

And finally, complete the kubelet configuration with (enable kubelet autostart):

```bash
$ sudo systemctl enable kubelet
```

_Excellent, you have a running cluster!_

But there needs to be one nicety added to this setup: an Ingress controller.

## Installing the Ingress controller

**The Ingress controller is necessary to read your Ingress manifests and route traffic inside the cluster.**

Kubernetes has no default Ingress controller, so you must install one if you wish to use it.

When you executed the `master.sh` command, it created a `traefik.yaml` file.

```bash
$ ls -1
config.yaml
flannel.yaml
master.sh
traefik.yaml
```

Traefik is an ingress controller and can be installed with the following command:

```bash
$ kubectl apply -f traefik.yaml
namespace/traefik created
serviceaccount/traefik created
clusterrole.rbac.authorization.k8s.io/traefik created
clusterrolebinding.rbac.authorization.k8s.io/traefik created
daemonset.apps/traefik created
```

The Ingress controller is deployed as a Pod, so you should wait until the image is downloaded.

If the installation was successful, you should be able to return the host and `curl` the first worker Node:

```bash
$ curl <ip minikube-m02>
curl: (7) Failed to connect to 192.168.49.3 port 80: Operation timed out
```

**Unfortunately, that IP address lives in the Docker network and is not reachable from your Mac or Windows (it's reachable if you are working on Linux).**

_But, worry not._

You can launch a jumpbox — a container with a terminal session in the same network:

```bash
$ docker run -ti --rm --network=minikube ghcr.io/learnk8s/netshoot:2023.03
```

From this container, you can reach any node of the cluster — let's retrieve and repeat the experiment:

```bash
$ curl <ip minikube-m02>
404 page not found
```

You should see a `404 page not found` message.

> `404 page not found` is not an error. This is a message from the Ingress controller saying no routes are set up for this URL.

_Is your cluster ready now?_

Yes, there's one minor step needed.

You have to be logged in to the control plane node to issue `kubectl` commands.

_Wouldn't it be easier if you could send commands from your computer instead?_

## Exposing the kubeconfig

The `kubeconfig` file holds the credentials to connect to the cluster.

Currently, the file is saved on the control plane node, but you can copy the content and save it on your computer (outside of the minikube virtual machine).

You can retrieve the content with:

```bash
$ minikube ssh --node minikube cat '$HOME/.kube/config' >kubeconfig
```

Now, the content is saved in your local file, named `kubeconfig`.

If you are on Mac or Windows, you should apply one small change: replace `<master-node-ip>:8443` with localhost and the correct port exposed by Docker.

First, list your nodes with:

```bash
$ docker ps
CONTAINER ID   IMAGE                                 PORTS                       NAMES
5717b8d142ac   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:53517->8443/tcp   minikube-m03
d5e1dbe9611c   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:53503->8443/tcp   minikube-m02
648efe712022   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:53486->8443/tcp   minikube
```

Find the port that forwards to 8443 for the control plane (in the above example is `53486`).

And finally, replace `<master-node-ip>:8443` in your kubeconfig:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CR…
    server: https://127.0.0.1:<insert-port-here>
  name: mk
contexts:
```

Finally, navigate to the directory where the file is located and execute the following line:

```bash
export KUBECONFIG="${PWD}/kubeconfig"
```

Or if you are using PowerShell:

```powershell
$Env:KUBECONFIG="${PWD}\kubeconfig"
```

You can verify that you are connected to the cluster with:

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:53486
CoreDNS is running at https://127.0.0.1:53486/api/v1/namespaces/kube…
```

> Please note that you should export the path to the `kubeconfig` whenever you create a new terminal session.

You can also store the credentials alongside the default `kubeconfig` file instead of changing environment variables.

However, since you will destroy the cluster at the end of this module, it's better to keep them separated for now.

> If you are still convinced you should merge the details with your kubeconfig, [you can find the instructions on how to do so here.](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#merging-kubeconfig-files)

**Congratulations!**

You just configured a fully functional Kubernetes cluster!

**Recap:**

- You created three virtual machines using minikube.
- You bootstrapped the Kubernetes control plane node using `kubeadm`.
- You installed Flannel as the network plugin.
- You installed an Ingress controller.
- You configured `kubectl` to work from outside the control plane.

The cluster is fully functional, and it's time to deploy an application.

## Deploying the application

You can find the application's YAML definition in `app.yaml`.

The file contains three Kubernetes resources:

- A deployment for the podinfo pod. It is currently set to a single replica, but you should deploy 2.
- There is a Service to route traffic to the pods.
- There is an Ingress manifest to route the traffic to the pod.

You can create the resources with

```bash
$ kubectl apply -f https://raw.githubusercontent.com/learnk8s/devopdays-sg-ha/refs/heads/main/app.yaml
```

**Verifying the deployment:**

List the current IP addresses for the cluster from your host machine with:

```bash
$ minikube node list
minikube      192.168.105.18
minikube-m02  192.168.105.19
minikube-m03  192.168.105.20
```

If your app is deployed correctly, you should be able to execute:

- `kubectl get pods -o wide` and see two pods deployed — one for each node.
- `curl <ip minikube-m02>` from the jumpbox and see the pod hostname in the JSON output.
- `curl <ip minikube-m03>` from the jumpbox and see the pod hostname in the JSON output.

If your deployment isn't quite right, try to [debug it using this handy flowchart.](https://learnk8s.io/troubleshooting-deployments)

## Testing resiliency

**Kubernetes is engineered to keep running even if some components are unavailable.**

So you could have a temporary failure to one the scheduler, but the cluster will still keep operating as usual.

The same is true for all other components.

The best way to validate this statement is to break the cluster.

_What happens when a Node becomes unavailable?_

_Can Kubernetes gracefully recover?_

_And what if the primary Node is unavailable?_

Let's find out.

### Making the nodes unavailable

Observe the nodes and pods in the cluster with:

```bash
$ watch kubectl get nodes,pods -o wide
NAME                STATUS   ROLES                  INTERNAL-IP
node/minikube       Ready    control-plane          192.168.105.18
node/minikube-m02   Ready    <none>                 192.168.105.19
node/minikube-m03   Ready    <none>                 192.168.105.20

NAME                               READY   STATUS   NODE
pod/hello-world-5d6cfd9db8-nn256   1/1     Running  minikube-m02
pod/hello-world-5d6cfd9db8-dvnmf   1/1     Running  minikube-m03
```

Observe how Pods and Nodes are in the "Running" and "Ready" states.

_Let's break a worker node and observe what happens._

In another terminal session, shut down the second worker node with:

```bash
$ minikube node stop minikube-m03
✋  Stopping node "minikube-m03"  ...
🛑  Successfully stopped node minikube-m03
```

**Please note the current time and set the alarm for 5 minutes** — (you will understand why soon).

Observe the node almost immediately transitioning to a "Not Ready" state:

```bash
$ kubectl get nodes -o wide
NAME                STATUS   ROLES                  INTERNAL-IP
node/minikube       Ready    control-plane          192.168.105.18
node/minikube-m02   Ready    <none>                 192.168.105.19
node/minikube-m03   NotReady <none>                 192.168.105.20
```

**The application should still serve traffic as usual.**

Try to issue a request from the jumpbox with:

```bash
$ curl <minikube-m02 IP address>
Hello, hello-world-5d6cfd9db8-nn256
```

However, there is something odd with the Pods.

_Have you noticed?_

```bash|command=1|title=bash
$ watch kubectl get pods -o wide
NAME                               READY   STATUS   NODE
pod/hello-world-5d6cfd9db8-nn256   1/1     Running  minikube-m02
pod/hello-world-5d6cfd9db8-dvnmf   1/1     Running  minikube-m03
```

_Why is the Pod on the second worker node still in the "Running" state?_

_And, even more puzzling, Kubernetes knows the Node is not available (e.g. it's "NotReady"), why isn't rescheduling the Pod?_

### Making the control plane unavailable

You should stop the control plane:

```bash
$ minikube node stop minikube
```

> Please note that, from this point onwards, you won't be able to observe the state of the cluster with kubectl.

Notice how the application still serves traffic as usual.

Execute the following command from the jumpbox:

```bash
$ curl <minikube-m02 IP address>
Hello, hello-world-5d6cfd9db8-nn256
```

**In other words, the cluster can still operate even if the control plane is unavailable.**

You won't be able to schedule or update workloads, though.

You should restart the control plane with:

```bash|command=1|title=bash
$ minikube node start minikube
```

Please notice that minikube may assign a different forwarding port to this container, and you might need to fix your kubeconfig file.

You can easily verify if the port has changed with:

```bash
CONTAINER ID   IMAGE                                 PORTS                       NAMES
5717b8d142ac   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:53517->8443/tcp   minikube-m03
d5e1dbe9611c   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:53503->8443/tcp   minikube-m02
648efe712022   gcr.io/k8s-minikube/kicbase:v0.0.37   127.0.0.1:65375->8443/tcp   minikube
```

In this case, the port used to be `53486` and now is `65375`.

You should amend your kubeconfig file accordingly:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CR…
    server: https://127.0.0.1:<insert-new-port-here>
  name: mk
contexts:
```

With this minor obstacle out of your way, halt the remaining worker Node with:

```bash
$ minikube node stop minikube-m02
✋  Stopping node "minikube-m02"  ...
🛑  Successfully stopped node minikube-m02
```

You should verify that both Nodes are in the _NotReady_ state:

```bash
$ kubectl get nodes -o wide
NAME                STATUS   ROLES                  INTERNAL-IP
node/minikube       Ready    control-plane          192.168.105.18
node/minikube-m02   NotReady <none>                 192.168.105.19
node/minikube-m03   NotReady <none>                 192.168.105.20
```

And the application is finally unreachable.

Neither `curl <minikube-m02 IP address>` nor `curl <minikube-m03 IP address>` from the jumpbox will work now.

### Scheduling workloads with no worker node

Despite not having any worker nodes, you can still scale the application to 5 instances:

```bash
$ kubectl edit deployment hello-world
deployment.apps/hello-world edited
```

And change the replicas to `replicas: 5`.

Monitor the pods with:

```bash
$ watch kubectl get pods -o wide
NAME                           READY   STATUS
hello-world-5d6cfd9db8-2k7f9   0/1     Pending
hello-world-5d6cfd9db8-8dpgd   0/1     Pending
hello-world-5d6cfd9db8-cwwr2   0/1     Pending
hello-world-5d6cfd9db8-dvnmf   1/1     Terminating
hello-world-5d6cfd9db8-nn256   1/1     Running
hello-world-5d6cfd9db8-rjd54   1/1     Running
```

The Pods stay pending because no worker node is available to run them.

In another terminal session, start both nodes with:

```bash
$ minikube node start minikube-m02
$ minikube node start minikube-m03
```

It might take a while for the two virtual machines to start, but, in the end, the Deployment should have five replicas "Running".

You can test that the application is available from the jumpbox with:

```bash
$ curl <minikube-m02 IP address>
Hello, hello-world-5d6cfd9db8-8dpgd
```

But, again, there's something odd.

_Have you noticed how the Pods are distributed in the cluster?_

Let's pay attention to the Pod distribution in the cluster:

```bash
$ kubectl get pods -o wide
NAME                           READY   STATUS    NODE
hello-world-5d6cfd9db8-2k7f9   1/1     Running   minikube-m02
hello-world-5d6cfd9db8-8dpgd   1/1     Running   minikube-m02
hello-world-5d6cfd9db8-cwwr2   1/1     Running   minikube-m02
hello-world-5d6cfd9db8-nn256   1/1     Running   minikube-m02
hello-world-5d6cfd9db8-rjd54   1/1     Running   minikube-m02
```

In this case, five Pods run on worker1 and none on worker2.

However, you might experience a slightly different distribution.

You could have any of the following:

- 5 Pods on worker1, 0 on worker2
- 0 Pods on worker1, 5 on worker2
- 3 Pods on worker1, 2 on worker2
- 2 Pods on worker1, 3 on worker2

And if you are lucky, you could also have:

- 4 Pods on worker1, 1 on worker2
- 1 Pods on worker1, 4 on worker2

_Why isn't Kubernetes rebalancing the Pods?_