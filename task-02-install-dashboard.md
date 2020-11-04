# Install The Kubernetes Dashboard

The Kubernetes dashabord can be installed by following the directions [here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) but that only allows for acvcess either:

- From within the cluster itselt
- From a machine with kubectl and by using the `proxy` command

The steps in this document willk install the dashboard and make it accessible via any web browers on the same LAN as the cluster

## Install

We will initially install the Kubernetes dashboard by applying the manifest and following the typical steps for accessing it.

First ensure the `kubectl` command works from another PC or laptop and can access and administer your cluster

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

The dashboard is deployed to its own namespace.  You can see the various components by running:

```
kubectl get all -n kubernetes-dashboard
```

> Output

```
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-7b59f7d4df-cpvr4   1/1     Running   0          44s
pod/kubernetes-dashboard-74d688b6bc-hnp6k        1/1     Running   0          44s

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.99.19.154   <none>        8000/TCP   44s
service/kubernetes-dashboard        ClusterIP   10.96.206.68   <none>        443/TCP    44s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           44s
deployment.apps/kubernetes-dashboard        1/1     1            1           44s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-7b59f7d4df   1         1         1       44s
replicaset.apps/kubernetes-dashboard-74d688b6bc        1         1         1       44s
```


## Create User To Access The Dashboard

These are the same as the [Creating Sample User](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md) instructions

Create a service account

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Create a ClusterRoleBinding

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Get a bearer token

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

> Output

```
Name:         admin-user-token-n8nhx
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: e7b6f7bc-fb10-42c8-b775-40c8106c2689

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NGVzL3Nl... [SNIP]...iaS04l2axbeqzuyX5Aug
```

The token will be a long string of alphanumeric characters.  Copy-paste the value


## Access the Dashboard via proxy

Run following command:

```
kubectl proxy
```

Now, form the same PC where you ran the above command, open the following URL in a web browser:

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Select `Token` and paste the token you received earlier into the `Enter token` field.  You should now have access to the dashboard

<img src="https://github.com/dleewo/kubernetes-bare-metal-tasks/raw/main/images/dashboard-login.png" width="600" border="1" />

Use CTRL-C to stop the proxy when done


## Access the Dashboard via Load Balancer

This step assumes you have installed and are using MetalLB or a similar load balancer solution for your cluster.  See [Install Load Balancer (MetalLB)](task-01-install-load-balancer.md) for help installing and configuring MetalLB

The dahsboard is exposed via a `ClusterIP` service as can be seen here:

```
$ kubectl get services -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.99.19.154   <none>        8000/TCP   15m
kubernetes-dashboard        ClusterIP   10.96.206.68   <none>        443/TCP    15m
```

We need to edit the serivce deplopyment.  Run

```
kubectl edit service kubernetes-dashboard -n kubernetes-dashboard
```

Towards the end, look for this section

```
spec:
  clusterIP: 10.96.206.68
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30675
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
```

  and edit the file to change the type to `LoadBalancer` so it should now look like this:

```
  spec:
  clusterIP: 10.96.206.68
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30675
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: LoadBalancer
```

If you now examine the service again, you will see that an `EXTERNAL-IP` has been assigned:

```
(base) Dereks-MacBook-Pro:.kube derek$ kubectl get services -n kubernetes-dashboard
NAME                        TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP      10.99.19.154   <none>          8000/TCP        18m
kubernetes-dashboard        LoadBalancer   10.96.206.68   192.168.20.85   443:30675/TCP   18m
```

You can access the dashboard using:

https://192.168.20.85

>WARNING: Chrome and Safari will not let you access this site, but FireFox does.  We will configure SSL in the next step


## Use Custom SSL Certificates

This assumes you have SSL certs.  If you have a domain, you can use [Let's Encrypt](https://letsencrypt.org/) to obtain certificates.  Place the files in the folder `$HOME/certs` (or change the following command to point to the correct location).  In my case, the certificate files are `privkey.pem` and `fullchain.pem`

The following will delete the existing secret and create a new one with the SSL certificates

```
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
```
Now we need to edit the deployment

```
kubectl edit deployment kubernetes-dashboard -n kubernetes-dashboard
```

And in the `args` to the container, add the `--tls-cert` and `--tls-key` arguments.  I removed the `--auto-generate-certificates` argumnent, but I believe it can be left in and it'll be used as a fallback.  Replace the arguments with your actual certificate filenames if they are different

```
    spec:
      containers:
      - args:
        - --tls-cert-file=/fullchain.pem
        - --tls-key-file=/privkey.pem
        - --namespace=kubernetes-dashboard
        image: kubernetesui/dashboard:v2.0.0
```


I use [CloudFlare's](https://www.cloudflare.com/) free service to manage the DNS for my `leewo.org` domain name.  I added an entry for `k8-dev-dashboard.leewo.org` to point to the IP address allocated by the MetalLB load balancer.  Once done, I can then access

https://k8s-dev-dashboard.leewo.org

and get proper SSL termination.  If you don't have a DNS provider you can always add an entry to your `hosts` file
