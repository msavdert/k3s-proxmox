# Chapter 6: Certificate Management (cert-manager)

Secure communication is a critical component of a production-grade Kubernetes cluster. In this chapter, we will install **cert-manager** to automate the issuance and renewal of TLS certificates, and configure our **Cilium Gateway** to support secure HTTPS traffic.

## 1. Install cert-manager

We will use Helm to install cert-manager. Starting with version 1.15, cert-manager supports the Kubernetes Gateway API, but it must be explicitly enabled.

> [!IMPORTANT]
> We are using version **v1.20.2** which is the latest stable release. We also enable the `config.enableGatewayAPI` flag to allow cert-manager to watch Gateway resources.

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io --force-update

# Check for the latest versions
helm search repo jetstack -l | head -n 5

# Install cert-manager with CRDs and Gateway API support
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.20.2 \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true
```

**Verify Installation:**
```bash
kubectl get pods -n cert-manager
```

### Optional: Install cmctl

`cmctl` is a command-line tool that can help you manage cert-manager resources and troubleshoot issues more easily.

```bash
# Install cmctl (macOS example)
brew install cert-manager/tap/cert-manager
```

## 2. Configure a ClusterIssuer

A **ClusterIssuer** defines how cert-manager should obtain certificates across the entire cluster. For our internal homelab (using `.local` domains), we will start with a **Self-Signed Issuer**.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF
```

> [!NOTE]
> Self-signed certificates are great for bootstrapping, but they will trigger browser warnings. In a later stage, you might replace this with an internal CA or Let's Encrypt (if using a public domain).

## 3. Enable HTTPS on Cilium Gateway

Now we will update our `cilium-gateway` to listen on port 443. By adding the `cert-manager.io/cluster-issuer` annotation, cert-manager will automatically generate a certificate based on the `tls` block in the listener.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
  namespace: kube-system
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "longhorn.local"
    tls:
      mode: Terminate
      certificateRefs:
      - name: longhorn-tls-secret
    allowedRoutes:
      namespaces:
        from: All
EOF
```

> [!TIP]
> If you are using traditional `Ingress` resources instead of the Gateway API, you can enable automatic certificate provisioning by adding the same annotation (`cert-manager.io/cluster-issuer`) to your Ingress metadata. This is known as **ingress-shim**.

## 4. Verification & Troubleshooting

Once the Gateway is updated, cert-manager should detect the new `tls` listener and create a `Certificate` resource, which in turn creates a Kubernetes `Secret`.

**Check the Certificate status:**
```bash
kubectl get certificate -n kube-system
```

**Troubleshoot (if not Ready):**
If the certificate status is not `Ready: True`, you can inspect the resource for errors:
```bash
kubectl describe certificate -n kube-system longhorn-tls-secret
```

**Check the generated Secret:**
```bash
kubectl get secret -n kube-system longhorn-tls-secret
```

### Testing the HTTPS Access

Try to access your service via HTTPS using `curl`:

```bash
# Use -k to ignore the self-signed certificate warning
curl -vk https://longhorn.local
```

You should see an HTTP 200 response from the Longhorn frontend.

> [!WARNING]
> Because we are using a self-signed certificate, your browser will show a security warning. You will need to click "Advanced" and "Proceed" to access the UI.

---

> [!TIP]
> If you decide to use a real domain later, you can simply change the `ClusterIssuer` to use Let's Encrypt and update the Gateway hostname. cert-manager will handle the transition automatically.
