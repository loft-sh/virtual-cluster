# Virtual Kubernetes Cluster

Virtual Kubernetes Cluster are fully functional Kubernetes clusters but they run inside a namespace of another cluster. Virtual Clusters are a part of [loft](https://loft.sh) (see [loft documentation](https://loft.sh/docs/vclusters/basics) for more information)

## Why Virtual Clusters?

Virtual clusters:

- are much cheaper than "real" clusters (shared resources and single virtual cluster pod)
- can be created and cleaned up again in seconds (great for CI/CD or testing)
- much more powerful than a simple namespace (virtual clusters allow users to use CRDs etc.)
- allow users to install apps which require cluster-wide permissions while being limited to actually just one namespace within the host cluster
- provide strict isolation while still allowing you to share certain services of the underlying host cluster (e.g. using the host's ingress-controller and cert-manager)

## How does it work?

The basic idea of a virtual cluster is to spin up an incomplete new Kubernetes cluster within an existing cluster and sync certain core resources between the two clusters to make the virtual cluster fully functional. The virtual cluster itself only consists of the core Kubernetes components: API server, controller manager and etcd. Besides some core kubernetes resources like pods, services, persistentvolumeclaims etc. that are needed for actual execution, all other kubernetes resources (like deployments, replicasets, resourcequotas, clusterroles, crds, apiservices etc.) are purely handled within the virtual cluster and NOT synced to the host cluster. This makes it possible to allow each user access to an complete own kubernetes cluster with full functionality, while still being able to separate them in namespaces in the actual host cluster.

- A virtual Kubernetes cluster in loft is tied to a single namespace. The virtual cluster and hypervisor run within a single pod that consists of two parts:
- a k3s instance which contains the Kubernetes control plane (API server, controller manager and etcd) for the virtual cluster
an instance of a virtual cluster hypervisor which is mainly responsible for syncing cluster resources between the k3s powered virtual cluster and the underlying host cluster

## Install

In order to install virtual cluster you need to find out the Service CIDR of your host cluster. This can be done by creating a service with a faulty ClusterIP in the host cluster:

```

```
