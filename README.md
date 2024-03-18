# Setup k8s cluster - raspberry-pi 

![Cluster Image](docs/cluster.jpg)

Hardware:
  - [6 x raspberry pic 4B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
  - [1 x 8-Port Gigabit Easy Smart Switch ](https://www.tp-link.com/in/business-networking/easy-smart-switch/tl-sg108e/v6/)
  - [1 x AC750 Wireless Travel Router](https://www.tp-link.com/in/home-networking/wifi-router/tl-wr902ac/)
  - [7 x 1FT CAT 6 Patch C Cables](https://www.amazon.in/gp/product/B005RCG0FK/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1)
  - [1 x Anker Power Port 6 (60W)](https://www.crazypi.com/raspberry-pi-cluster-power-supply-60w)
  - [Raspberry Pi Cluster 6 Layer Rack](https://www.crazypi.com/raspberry-pi-cluster-6-layer?search=cluster&description=true)
  - [Hoteon Power Strip plus + Cable Management Box](https://www.amazon.in/gp/product/B094NDJGYL/ref=ppx_yo_dt_b_asin_title_o03_s00?ie=UTF8&psc=1)

### Architecture: 
  
  https://miro.com/app/board/uXjVPcUEQXI=/?share_link_id=642944915070
  
### 1. Install k3s
```shell
ansible-playbook -i hosts playbook/k3s/k3s.yml
```

### 2. Install metallb

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
kubectl apply -f apps/metallb/config.yaml
```

### 3. install Ingress-nginx

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

```shell
edit ingress-nginx-controller 
replace NodePort to LoadBalancer
```

### 4. Install cert-manager
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

#### Setup SSL cert issuer 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
data:
  api-token: <token>
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <your-email>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          cloudflare:
            email: <your-email>
            apiKeySecretRef:
              name: cloudflare-api-token-secret
              key: api-token
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <your-email>
spec:
  acme:
    email: <your-email>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-stating
    solvers:
      - dns01:
          cloudflare:
            email: <your-email>
            apiKeySecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

### Setup Longhorn:

```shell
ansible-playbook -i hosts playbook/longhorn/longhorn.yaml

kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```
### Blogs:

  - https://banzaicloud.com/blog/load-balancing-on-prem/
  - https://fredrickb.com/2021/08/22/recreating-the-raspberry-pi-homelab-with-kubernetes/
  - https://tothecloud.dev/using-certmanager-with-cloudflare-and-kubernetes/


helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack

### Deploy Metrics Server
```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml           
```

### Deploy Dashboard
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

kubectl apply -f apps/dashboard/service-account.yml

kubectl create token admin-user -n  kubernetes-dashboard
```
