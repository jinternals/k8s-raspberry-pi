# Setup k8s cluster - raspberry-pi 

![Cluster Image](docs/cluster.jpg)

Hardware:
  - [6 x raspberry pic 4B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
  - [1 x Omada ER7212PC](https://www.omadanetworks.com/in/business-networking/omada-router-integrated-router/er7212pc/) (gateway + switch in one: routes the cluster LAN and connects all nodes)
  - [1 x AC750 Wireless Travel Router](https://www.tp-link.com/in/home-networking/wifi-router/tl-wr902ac/) (client mode: Wi-Fi uplink to the home network for the ER7212PC WAN port)
  - [7 x 1FT CAT 6 Patch C Cables](https://www.amazon.in/gp/product/B005RCG0FK/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1)
  - [1 x Anker Power Port 6 (60W)](https://www.crazypi.com/raspberry-pi-cluster-power-supply-60w)
  - [Raspberry Pi Cluster 6 Layer Rack](https://www.crazypi.com/raspberry-pi-cluster-6-layer?search=cluster&description=true)
  - [Hoteon Power Strip plus + Cable Management Box](https://www.amazon.in/gp/product/B094NDJGYL/ref=ppx_yo_dt_b_asin_title_o03_s00?ie=UTF8&psc=1)

### Architecture: 
  
  https://miro.com/app/board/uXjVPcUEQXI=/?share_link_id=642944915070
  
### Network topology

```
Home router (Wi-Fi)
      │  Wi-Fi
TL-WR902AC (client mode)
      │  Ethernet
ER7212PC WAN ── gateway + switch ── cluster LAN 192.168.0.0/24
                                     ├── k3s server  192.168.0.100
                                     ├── k3s agents  192.168.0.101-106
                                     └── MetalLB IPs 192.168.0.200-240
```

The cluster LAN sits behind the ER7212PC (NAT). Devices on the home network reach cluster services
only through port forwards on the ER7212PC WAN.

### Network assumptions

Adjust these to your network before following the steps:

| Item | Value used here | Where to change |
|---|---|---|
| Cluster LAN | `192.168.0.0/24` (nodes `.100`–`.106`) | [hosts](hosts) |
| MetalLB IP pool | `192.168.0.200`–`192.168.0.240` (keep it outside your router's DHCP range) | [apps/metallb/config.yaml](apps/metallb/config.yaml) |
| Public Gateway IP | first IP from the pool (`192.168.0.200`) | assigned by MetalLB |
| Internal Gateway IP | `192.168.0.201` | [apps/internal-gateway/gateway.yaml](apps/internal-gateway/gateway.yaml) |
| Domain | `homelabs.co.in` (DNS on Cloudflare) | [apps/nginx-gateway-demo/app.yaml](apps/nginx-gateway-demo/app.yaml) |

### 1. Install k3s
```shell
ansible-playbook -i hosts playbook/k3s/k3s.yml
```

The playbook saves the cluster kubeconfig to `~/Installed/k8s/pi/config`, pointing at the first k3s server:
```shell
export KUBECONFIG=~/Installed/k8s/pi/config
kubectl get nodes
```

### 2. Install metallb

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
kubectl apply -f apps/metallb/config.yaml
```

### 3. Install Gateway API with NGINX Gateway Fabric

Tested with NGINX Gateway Fabric v2.7.2 (bundles Gateway API v1.6.1).

```shell
# Gateway API CRDs (version pinned by NGF)
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.7.2" | kubectl apply -f -

# NGINX Gateway Fabric CRDs
kubectl apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/crds.yaml

# NGINX Gateway Fabric controller + `nginx` GatewayClass
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/default/deploy.yaml
```

Verify:
```shell
kubectl get pods -n nginx-gateway
kubectl get gatewayclass nginx
```

Each `Gateway` you create gets its own `LoadBalancer` Service in the Gateway's namespace, and MetalLB assigns it an IP from `apps/metallb/config.yaml`:
```shell
kubectl get svc -A | grep LoadBalancer
```
To expose it outside the cluster LAN, port-forward TCP 80/443 on the ER7212PC WAN to that IP. To reach it from the internet,
also forward 80/443 on the home router to the ER7212PC WAN IP (see [Network topology](#network-topology)).
See [nginx.md](nginx.md) for a demo Gateway + HTTPRoute.

### 4. Install cert-manager

Install cert-manager **after** the Gateway API CRDs (step 3):
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.2/cert-manager.yaml
kubectl get pods -n cert-manager   # wait for cert-manager, cainjector and webhook to be Running
```

#### Enable Gateway API support

**Why:** the Gateway in [nginx.md](nginx.md) gets its TLS certificate from the
`cert-manager.io/cluster-issuer` annotation. cert-manager only acts on that annotation when it runs with
`--enable-gateway-api`, which the static `cert-manager.yaml` does not set. Without it, the HTTPS listener's
secret is never created and HTTPS does not work.

**Enable it** (adds the flag to the controller's args; skipped if already present, because a repeated patch appends the flag again):
```shell
kubectl -n cert-manager get deploy cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | grep -q -- --enable-gateway-api \
|| kubectl -n cert-manager patch deployment cert-manager --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-gateway-api"}]'

kubectl -n cert-manager rollout status deploy/cert-manager
```

**Verify:**
```shell
kubectl -n cert-manager logs deploy/cert-manager | grep -i "gateway api"
# expected: "enabling the sig-network Gateway API certificate-shim and HTTP-01 solver"
```

**After upgrading cert-manager:** re-applying `cert-manager.yaml` resets the args and removes the flag. Run the
enable command above again after every upgrade. To avoid that, install with Helm, which keeps the setting:
```shell
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager --version v1.21.2 \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true --set config.gatewayAPI.enabled=true
```

**Skipping it:** if you'd rather not enable it, remove the annotation from the Gateway and create a
`Certificate` for each HTTPS hostname yourself (`secretName` must match the listener's `certificateRefs`).

#### Setup SSL cert issuer 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
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
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: <your-email>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

### Setup Longhorn:

```shell
ansible-playbook -i hosts playbook/longhorn/longhorn.yaml

kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.12.1/deploy/longhorn.yaml
```

The playbook installs `open-iscsi`, `nfs-common`, `cryptsetup`, `dmsetup`, loads `iscsi_tcp`/`dm_crypt` and starts `iscsid`. First start is slow while the Pis pull images.

Longhorn UI (no login, local only) on the LAN-only internal Gateway:
```shell
kubectl apply -f apps/internal-gateway/gateway.yaml -f apps/longhorn/httproute.yaml
```
- From the cluster LAN: `http://192.168.0.201:8081`
- From another network behind the same router: port-forward TCP `8081` → `192.168.0.201:8081` on the router in front of the cluster, then open `http://<router IP>:8081`.

Never expose this port to the internet.

### Blogs:

  - https://banzaicloud.com/blog/load-balancing-on-prem/
  - https://fredrickb.com/2021/08/22/recreating-the-raspberry-pi-homelab-with-kubernetes/
  - https://tothecloud.dev/using-certmanager-with-cloudflare-and-kubernetes/

### Deploy Prometheus
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```

### Deploy Metrics Server
```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.9.0/components.yaml
```

### Deploy Headlamp (dashboard)

The Kubernetes Dashboard project is archived; [Headlamp](https://headlamp.dev) is its replacement.
```shell
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update
helm install headlamp headlamp/headlamp --namespace kube-system --version 0.45.0

kubectl -n kube-system create serviceaccount headlamp-admin
# The chart already creates a ClusterRoleBinding named "headlamp-admin" (for its own SA), so use another name
kubectl create clusterrolebinding headlamp-admin-user --clusterrole=cluster-admin --serviceaccount=kube-system:headlamp-admin

# Login token
kubectl create token headlamp-admin -n kube-system

# Open http://localhost:8080
kubectl port-forward -n kube-system service/headlamp 8080:80
```

To reach it without port-forward, expose it over plain HTTP on its own LAN-only Gateway (`192.168.0.201`):
```shell
kubectl apply -f apps/internal-gateway/gateway.yaml -f apps/headlamp/httproute.yaml
```
- From the cluster LAN: `http://192.168.0.201`
- From another network behind the same router: port-forward TCP `8080` → `192.168.0.201:80` on the router in front of the cluster, then open `http://<router IP>:8080`.

Never expose this port to the internet.

The login token is sent unencrypted, so only use this on a trusted network.
