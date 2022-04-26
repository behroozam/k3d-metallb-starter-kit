# Overview

Setup and Teardown a local Kubernetes Cluster with a Load Balancer, so that you can deploy to a local environment for local development.

# Prerequisites

- docker - https://docs.docker.com/get-docker/
- k3d - (v5.0.0) - https://github.com/rancher/k3d/releases
- jq - https://stedolan.github.io/jq/
- kubectls - https://kubernetes.io/docs/tasks/tools/

# Setup

Create the Cluster and validate it's creation:

```bash
#create config for k3d cluster

cat <<EOF>> cluster1.yml
---
apiVersion: k3d.io/v1alpha3
kind: Simple
name: cluster-1
kubeAPI:
 hostIP: "127.0.0.1"
 hostPort: "6445"

options:
 k3s: # options passed on to K3s itself
   extraArgs: # additional arguments passed to the `k3s server|agent` command; same as `--k3s-arg`
     - arg: --disable=traefik
       nodeFilters:
         - server:*
     - arg: --disable=servicelb
       nodeFilters:
         - server:*

# create the k3d cluster
k3d cluster create --config cluster1.yml


# validate the cluster master and worker nodes
kubectl get nodes
```

Deploy the Load Balancer:

```bash

for cluster_name in $(docker network list --format "{{ .Name}}" | grep k3d); do

cidr_block=$(docker network inspect $cluster_name | jq '.[0].IPAM.Config[0].Subnet' | tr -d '"')
cidr_base_addr=${cidr_block%???}
ingress_first_addr=$(echo $cidr_base_addr | awk -F'.' '{print $1,$2,255,0}' OFS='.')
ingress_last_addr=$(echo $cidr_base_addr | awk -F'.' '{print $1,$2,255,255}' OFS='.')
ingress_range=$ingress_first_addr-$ingress_last_addr

# switch context to current cluster
kubectl config use-context $cluster_name

# deploy metallb
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml

# configure metallb ingress address range
cat <<EOF | kubectl apply -f -
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
      - $ingress_range
EOF

# install nginx-ingress

helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.ingressClassResource.default="true"

done

```

# Validation

Create an Nginx test deployment and expose via a Load Balancer. If the Load Balancer is working correctly, an external ip address should be assigned by Metallb.

```bash
# create a deployment (i.e. nginx)
kubectl create deployment nginx --image=nginx

# expose the deployments using a LoadBalancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# obtain the ingress external ip
external_ip=$(k get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# test the loadbalancer external ip
curl $external_ip
```

Expected Output:

```bash
# expected output:

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# Teardown

Destroy the cluster

```bash
k3d cluster delete local-k8s
```
