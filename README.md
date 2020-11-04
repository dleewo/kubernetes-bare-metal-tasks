# Kubernetes - Bare Metal Tasks

After experimenting with building kuberneets clusters via various mechanisms:

* [Kubernetes The Hard Way - On Bare Metal](https://github.com/dleewo/kubernetes-the-hard-way-bare-metal)
* [Install kubernetes Using kubeadm](https://github.com/dleewo/kubernetes-install-via-kubeadm)

The next step is to confiugre and work with the cluster.  This repositry douments some of these steps.  Many of these tasks are applicbale for any cluster, but some will be unique to bare-metal of VM on-premise installs, for example, adding a load balancer.  Adding a load balancer is not needed for hosted clusters like Amazon EKS, but by adding one locally, you can deploy applkicatiosn in a similar manner to hosted clusters

## Tasks

* [Install Load Balancer (MetalLB)](task-01-install-load-balancer.md)
* [Install The Kubernetes Dashboard](task-02-install-dashboard.md)
