
```sh
# Start minikube witn no driver for bare metal/vm
minikube start --memory 4096 --driver=none

kubectl create secret generic consul-gossip-encryption-key --from-literal=key=$(consul keygen)

# Add helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && \
helm repo add grafana https://grafana.github.io/helm-charts && \
helm repo add hashicorp https://helm.releases.hashicorp.com && \
helm repo update

# Install Consul
helm install -f consul_values.yaml consul hashicorp/consul --version "0.27.0" --wait

# Take note of the ACL Bootstrap Token
kubectl get secrets/consul-bootstrap-acl-token --template={{.data.token}} | base64 -d  
# 1a369cec-14c9-3605-2d97-1bf80165f796
# And set it as env var for further use of consul command 
export CONSUL_HTTP_TOKEN=$(kubectl get secrets/consul-bootstrap-acl-token --template={{.data.token}} | base64 -d)

# get the ca.pem from the secret
kubectl get secret consul-ca-cert -o jsonpath="{.data['tls\.crt']}" | base64 --decode > ca.pem

# intentions
consul intention create -ca-file ca.pem -deny "*" "*"


# Fetch learning material
git clone https://github.com/hashicorp/learn-consul-kubernetes.git
cd learn-consul-kubernetes


# Deploy Prometheus
helm install -f layer7-observability/helm/prometheus-values.yaml prometheus prometheus-community/prometheus --version "11.7.0" --wait
# Deploy Grafana
helm install -f layer7-observability/helm/grafana-values.yaml grafana grafana/grafana --version "5.3.6" --wait

# Get grafana initial admin password
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# Grafana server internal dns
grafana.default.svc.cluster.local

# Prometeus push Gateway on port 9091
prometheus-pushgateway.default.svc.cluster.local 

# Expose Consul
kubectl apply -f expose_consul.yaml 
# Expose Grafana
kubectl apply -f expose_grafana.yaml 


# Deploy Apps
for i in web.yaml web2.yaml api.yaml; do
kubectl apply -f $i
done

for i in web.yaml web2.yaml api.yaml; do
kubectl delete -f $i
done


kubectl apply -f layer7-observability/hashicups
```