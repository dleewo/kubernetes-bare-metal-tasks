# Install Load Balancer - MetalLB

[MetalLB](https://metallb.universe.tf/) is a load-balancer implementation for bare metal Kubernetes clusters.

To install, apply the following manifests

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

The last secrte is only ever done the first time you are installing.  If you ever delete MetalLB and then re-add it, just deploy the first two manifests

