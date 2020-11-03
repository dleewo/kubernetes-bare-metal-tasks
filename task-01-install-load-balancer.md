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

MetalLB needs one or more IP address ranges.  Then, when a service specifies a type of `LoadBalancer`, MetalLB will assign one of those IP addresses thus making your application accessible externally.  The IP address range is set in a `ConfigMap`

Create the manifest as follows.  Be sure to set the IP address range to be used.  In my case, I am using a small range with just 5 IP addresses since this is only for a dev system:

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

We'll deploly `nginx`

```
kubectl create deployment nginx --image=nginx
```

Then expose a service of type `LoadBalancer`

```
kubectl expose deployment nginx --port=80 --target-port=80 --name=nginx --type=LoadBalancer
```

Let's view the service:

```
kubectl get services
```

> Output

```
kubernetes   ClusterIP      10.96.0.1       <none>          443/TCP        7h24m
nginx        LoadBalancer   10.105.65.113   192.168.20.85   80:32262/TCP   24s
```
As you can see, the `nginx` service was assigned IP address `192.168.20.85`.  Accessing that on a web browser should display the `nginx` welcome page









