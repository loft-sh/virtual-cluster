# Virtual Kubernetes Cluster

Virtual Kubernetes Clusters are fully functional Kubernetes clusters but they run inside a namespace of another cluster. Virtual Clusters are a part of [loft](https://loft.sh) (see [loft documentation](https://loft.sh/docs/vclusters/basics) for more information)

## Why Virtual Clusters?

Virtual clusters:

- are much cheaper than "real" clusters (shared resources and single virtual cluster pod)
- can be created and cleaned up again in seconds (great for CI/CD or testing)
- much more powerful than a simple namespace (virtual clusters allow users to use CRDs etc.)
- allow users to install apps which require cluster-wide permissions while being limited to actually just one namespace within the host cluster
- provide strict isolation while still allowing you to share certain services of the underlying host cluster (e.g. using the host's ingress-controller and cert-manager)

## How does it work?

The basic idea of a virtual cluster is to spin up an incomplete new Kubernetes cluster within an existing cluster and sync certain core resources between the two clusters to make the virtual cluster fully functional. The virtual cluster itself only consists of the core Kubernetes components: API server, controller manager and etcd. Besides some core kubernetes resources like pods, services, persistentvolumeclaims etc. that are needed for actual execution, all other kubernetes resources (like deployments, replicasets, resourcequotas, clusterroles, crds, apiservices etc.) are purely handled within the virtual cluster and NOT synced to the host cluster. This makes it possible to allow each user access to an complete own kubernetes cluster with full functionality, while still being able to separate them in namespaces in the actual host cluster.

- A virtual Kubernetes cluster is tied to a single namespace. The virtual cluster and hypervisor run within a single pod that consists of two parts:
- a k3s instance which contains the Kubernetes control plane (API server, controller manager and etcd) for the virtual cluster
an instance of a virtual cluster hypervisor which is mainly responsible for syncing cluster resources between the k3s powered virtual cluster and the underlying host cluster

![virtual cluster architecture](https://loft.sh/docs/media/ui/vclusters/vcluster-architecture.png)

## Install

For a one click solution take a look at [loft](https://loft.sh). For the manual install process without loft follow the guide below.

### Find out host cluster Service CIDR

In order to install virtual cluster you need to find out the Service CIDR of your host cluster. This can be done by creating a service with a faulty ClusterIP in the host cluster:

```
apiVersion: v1
kind: Service
metadata:
  name: faulty-service
spec:
  clusterIP: 1.1.1.1
  ports:
  - port: 80
    protocol: TCP
```

Then create the service via kubectl:

```
kubectl apply -f mysecret.yaml
The Service "faulty-service" is invalid: spec.clusterIP: Invalid value: "1.1.1.1": provided IP is not in the valid range. The range of valid IPs is 10.96.0.0/12
```

The error message shows the correct Service CIDR of the cluster, in this case `10.96.0.0/12`

### Install Virtual Cluster via helm

To start a new virtual cluster in any given namespace, you can use helm.

Create a values.yaml depending on your host cluster kubernetes version:
<details>
<summary><b>Host Kubernetes v1.16</b></summary>
<br>
  
```
virtualCluster:
  image: rancher/k3s:v1.16.13-k3s1
  extraArgs:
    - --service-cidr=10.96.0.0/12 # THE CLUSTER SERVICE CIDR HERE
  baseArgs:
    - server
    - --write-kubeconfig=/k3s-config/kube-config.yaml
    - --data-dir=/data
    - --no-deploy=traefik,servicelb,metrics-server,local-storage
    - --disable-network-policy
    - --disable-agent
    - --disable-scheduler
    - --disable-cloud-controller
    - --flannel-backend=none
    - --kube-controller-manager-arg=controllers=*,-nodeipam,-nodelifecycle,-persistentvolume-binder,-attachdetach,-persistentvolume-expander,-cloud-node-lifecycle
storage:
  size: 5Gi

# If you don't want to sync ingresses from the vCluster to 
# the host cluster uncomment the next lines
#syncer:
#  extraArgs: ["--disable-sync-resources=ingresses"]
```

</details>

<details>
<summary><b>Host Kubernetes v1.17</b></summary>
<br>

```
virtualCluster:
  image: rancher/k3s:v1.17.9-k3s1
  extraArgs:
    - --service-cidr=10.96.0.0/12 # THE CLUSTER SERVICE CIDR HERE
  baseArgs:
    - server
    - --write-kubeconfig=/k3s-config/kube-config.yaml
    - --data-dir=/data
    - --no-deploy=traefik,servicelb,metrics-server,local-storage
    - --disable-network-policy
    - --disable-agent
    - --disable-scheduler
    - --disable-cloud-controller
    - --flannel-backend=none
    - --kube-controller-manager-arg=controllers=*,-nodeipam,-nodelifecycle,-persistentvolume-binder,-attachdetach,-persistentvolume-expander,-cloud-node-lifecycle
storage:
  size: 5Gi

# If you don't want to sync ingresses from the vCluster to 
# the host cluster uncomment the next lines
#syncer:
#  extraArgs: ["--disable-sync-resources=ingresses"]
```

</details>

<details>
<summary><b>Host Kubernetes v1.18</b></summary>
<br>

```
virtualCluster:
  image: rancher/k3s:v1.18.6-k3s1
  extraArgs:
    - --service-cidr=10.96.0.0/12 # THE CLUSTER SERVICE CIDR HERE
storage:
  size: 5Gi

# If you don't want to sync ingresses from the vCluster to 
# the host cluster uncomment the next lines
#syncer:
#  extraArgs: ["--disable-sync-resources=ingresses"]
```

</details>

Then run:
```
helm install virtualcluster virtualcluster --repo https://charts.devspace.sh/ \
  --namespace virtualcluster \
  --values values.yaml \
  --create-namespace \
  --wait
```

### Connect to the virtual cluster

After installing the virtual cluster, make sure the virtual-cluster is running via kubectl:

```
kubectl get po -n virtualcluster
NAME                                                    READY   STATUS    RESTARTS   AGE
coredns-d798c9dd-kq64p-x-kube-system-x-virtualcluster   1/1     Running   0          3d17h
virtualcluster-0                                        2/2     Running   0          3d17h
```

Retrieve the kubeconfig via kubectl:
```
kubectl exec virtualcluster-0 --namespace virtualcluster -c syncer -- cat /root/.kube/config > kubeconfig.yaml
```

Forward the virtual cluster api port to localhost:
```
kubectl port-forward test-0 -n virtualcluster 8443:8443
Forwarding from 127.0.0.1:8443 -> 8443
Forwarding from [::1]:8443 -> 8443
```

Now you can access the virtual cluster via kubectl:
```
kubectl --kubeconfig kubeconfig.yaml get namespaces
NAME              STATUS   AGE
default           Active   83s
kube-system       Active   83s
kube-public       Active   83s
kube-node-lease   Active   83s
```

Congratulations, you now have a fully functional virtual kubernetes cluster!

## Delete the virtual cluster

To delete a virtual cluster and its created resources in the host cluster, you just have to delete the helm chart:
```
helm delete virtualcluster -n virtualcluster
```
