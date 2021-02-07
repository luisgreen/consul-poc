# consul-poc

## Purpose

Havin a Minikube environment to test a service mesh

## Prerequisites

- minikube

```sh
# Needed no driver for allowing internet access
$ sudo apt-get install conntrack socat
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && \
$ helm repo add grafana https://grafana.github.io/helm-charts && \
$ helm repo add hashicorp https://helm.releases.hashicorp.com && \
$ helm repo update
```

## Start Minikube 
```sh
$ minikube start --memory 4096 --driver=none
```

## Dashboard (if needed)

```sh
$ minikube dashboard
$ kubectl proxy --address='0.0.0.0' --disable-filter=true
```

## Consul Service Mesh

```sh
# usar  consul-values.yaml de las notas

$ helm install -f consul-values.yaml hashicorp hashicorp/consul --version "0.27.0" --wait
$ helm install -f layer7-observability/helm/prometheus-values.yaml prometheus prometheus-community/prometheus --version "11.7.0" --wait
$ helm install -f layer7-observability/helm/grafana-values.yaml grafana grafana/grafana --version "5.3.6" --wait

$ kubectl get services
NAME                                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                                   AGE
kubernetes                              ClusterIP   10.43.0.1             443/TCP                                                                   22m
hashicorp-consul-server                 ClusterIP   None                  8500/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP,8600/UDP   15m
hashicorp-consul-connect-injector-svc   ClusterIP   10.43.2.103           443/TCP                                                                   15m
hashicorp-consul-dns                    ClusterIP   10.43.22.2            53/TCP,53/UDP                                                             15m
hashicorp-consul-ui                     ClusterIP   10.43.85.85           80/TCP                                                                    15m

# If you only want consul by proxy
$ kubectl port-forward service/hashicorp-consul-ui 18500:80 --address 0.0.0.0
# Go to http://<ip>:18500

# To expose consul UI
$ kubectl apply -f expose_consul.yaml
# Go to http://<ip>:30000
```


Get your 'admin' user password by running:

```sh
$ kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

The Grafana server can be accessed via port 80 on the following DNS name from within your cluster `grafana.default.svc.cluster.local`

Get the Grafana URL to visit by running these commands in the same shell:

```sh
$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

$ kubectl --namespace default port-forward --address 0.0.0.0 $POD_NAME 3000
```

3. Login with the password from step 1 and the username: admin

## Creating intentions for traffic supervision intraservices

Service intent are filtered by container name, so note the section `spec.template.spec.containers.name`

```yaml
spec:
#...
  template:
#...
    spec:
      containers:
        - name: web # This containers called "web" inside deployments.
```

```sh
$ kubectl exec -it hashicorp-consul-server-0 -- /bin/sh

/ $ consul intention create -deny "*" "*"

# Check all services are denied by default
/ $ consul intention check web api
Denied

# Allow web to interact with api
/ $ consul intention create -allow web api
Created: web => api (allow)

/ $ consul intention check web api
Allowed
```

## Demo service Mesh

```sh
# Common Api Service
$ kubectl apply -f api.yaml

$ kubectl apply -f web.yaml
$ kubectl apply -f web2.yaml

# Go and open http://<ip>:30080/ui/ to see web.yaml deployment
# Go and open http://<ip>:30080/ui/ to see web2.yaml deployment
```

