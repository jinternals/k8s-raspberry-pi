# Steps for Enabling Nginx - Gateway API support

Tested with NGINX Gateway Fabric v2.7.2 (bundles Gateway API v1.6.1) and cert-manager v1.21.2.

### 1. Install Gateway API CRDs
```shell
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.7.2" | kubectl apply -f -
```

### 2. Install cert-manager with Gateway API support
Follow [README step 4](README.md#4-install-cert-manager), including **Enable Gateway API support**
(required for the `cert-manager.io/cluster-issuer` annotation on the Gateway below).

Then create the `letsencrypt-prod` / `letsencrypt-staging` ClusterIssuers from the [README](README.md#setup-ssl-cert-issuer).

### 3. Install the NGINX Gateway Fabric CRDs
```shell
kubectl apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/crds.yaml
```

### 4. Deploy the NGINX Gateway Fabric controller and `nginx` GatewayClass
```shell
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/default/deploy.yaml
```

### 5. Demo Application

Ready to deploy (uses `letsencrypt-staging`):
```shell
kubectl apply -f apps/nginx-gateway-demo/app.yaml
```

> **Note**: With Gateway API support enabled (step 2), the `cert-manager.io/cluster-issuer` annotation on the
> `Gateway` makes cert-manager create the certificate for each HTTPS listener, using the listener `hostname`
> and storing it in the secret named in `certificateRefs`. Don't also create a `Certificate` for the same secret.
>
> Use `letsencrypt-staging` until everything works, then switch to `letsencrypt-prod`.

Replace `homelabs.co.in` with your own domain (managed by the Cloudflare account used by the ClusterIssuers).

The manifest in [apps/nginx-gateway-demo/app.yaml](apps/nginx-gateway-demo/app.yaml) contains:
- `nginx-demo` namespace, `demo-deployment` (echo-server) and `demo-svc`
- `k8s-gateway`: HTTP listener for `*.homelabs.co.in` and HTTPS listener for `secure.homelabs.co.in`
  (certificate `secure-homelabs-cert` issued via the `cert-manager.io/cluster-issuer` annotation)
- HTTPRoutes: plain HTTP → demo, HTTP → HTTPS redirect for `secure.homelabs.co.in`, HTTPS → demo

#### Switch to production certificates
Once HTTPS works with the staging certificate:
```shell
# in apps/nginx-gateway-demo/app.yaml: letsencrypt-staging -> letsencrypt-prod
kubectl delete secret secure-homelabs-cert -n nginx-demo
kubectl apply -f apps/nginx-gateway-demo/app.yaml
```

### 6. Verify
```shell
kubectl get gateway,httproute -n nginx-demo
kubectl get certificate -n nginx-demo

# External IP assigned by MetalLB – forward 80/443 on the router(s) to it
kubectl get svc -n nginx-demo
```
