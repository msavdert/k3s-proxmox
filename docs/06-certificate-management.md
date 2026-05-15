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

## 3. Next Steps

Now that we have **cert-manager** and a **ClusterIssuer** ready, we can automate the issuance of certificates for any service in our cluster.

In **Chapter 8**, we will configure the **Cilium Gateway** to use this issuer and provide secure HTTPS access to our internal services like the Longhorn UI and Hubble UI.

---

> [!TIP]
> If you decide to use a real domain later, you can simply change the `ClusterIssuer` to use Let's Encrypt. cert-manager will handle the transition automatically.
