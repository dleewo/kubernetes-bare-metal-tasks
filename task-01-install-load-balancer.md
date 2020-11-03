# Install Load Balancer - MetalLB

[MetalLB](https://metallb.universe.tf/) is a load-balancer implementation for bare metal Kubernetes clusters.


## Install

To install, apply the following manifests

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

The last secret is only ever done the first time you are installing.  If you ever delete MetalLB and then re-add it, just deploy the first two manifests

MetalLB needs one or more IP address ranges.  Then, when a service specifies a tyep of `LoadBalancer`, MetalLB will assign one of those IP address thus making your application accessible externally.  The IP address range is set in a `ConfigMap`

Create the manifest as follows.  Be sure to set the IP address range to be used:

```
cat <<EOF | tee metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.20.85-192.168.20.89
EOF
```

Deploy the manifest

```
kubectl apply -f  metallb-config.yaml
```

## Smoke Test


