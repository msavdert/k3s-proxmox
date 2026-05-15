# Chapter 8: Exposing Services (Cilium Gateway API)

Now that we have **cert-manager** for certificates and **Longhorn** for storage, it's time to expose our internal services to our local network securely. We will use the **Cilium Gateway API** to provide a single entry point for HTTPS traffic.

## 1. Create the Cluster Gateway

The **Gateway** resource defines the infrastructure that listens for incoming traffic. We will configure it to handle both HTTP (port 80) and HTTPS (port 443).

> [!IMPORTANT]
> The `cert-manager.io/cluster-issuer` annotation tells cert-manager to automatically manage certificates for any `tls` listeners defined in this Gateway.

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
    hostname: "*.local"
    tls:
      mode: Terminate
      certificateRefs:
      - name: wildcard-local-tls
    allowedRoutes:
      namespaces:
        from: All
EOF
```

**Verify the Gateway IP:**
Cilium will assign an IP from the `local-pool` we configured in Chapter 5.

```bash
kubectl get gateway -n kube-system
```
*Note: Wait until the `ADDRESS` field is populated.*

## 2. Expose Longhorn UI

We use an **HTTPRoute** to map a hostname (`longhorn.local`) to the internal Longhorn service.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: longhorn-route
  namespace: longhorn-system
spec:
  parentRefs:
  - name: cilium-gateway
    namespace: kube-system
  hostnames:
  - "longhorn.local"
  rules:
  - backendRefs:
    - name: longhorn-frontend
      port: 80
EOF
```

## 3. Expose Hubble UI

Similarly, we can expose the Hubble network observability dashboard.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hubble-route
  namespace: kube-system
spec:
  parentRefs:
  - name: cilium-gateway
    namespace: kube-system
  hostnames:
  - "hubble.local"
  rules:
  - backendRefs:
    - name: hubble-ui
      port: 80
EOF
```

## 4. Verification

### Verify HTTPRoutes

Ensure that both routes are created and bound to the correct hostnames.

```bash
kubectl get httproute -A
```

Expected output:
```text
NAMESPACE         NAME             HOSTNAMES            AGE
kube-system       hubble-route     ["hubble.local"]     53s
longhorn-system   longhorn-route   ["longhorn.local"]   62s
```

### Check Certificate Status
Cert-manager should have detected the Gateway listener and created a secret.

```bash
kubectl get certificate -n kube-system wildcard-local-tls
```

### Local DNS Setup (/etc/hosts)
Since we are using `.local` domains, you need to map them to your Gateway IP in your machine's `/etc/hosts` file.

```bash
# Replace <GATEWAY_IP> with the address from 'kubectl get gateway'
sudo echo "<GATEWAY_IP> longhorn.local hubble.local" >> /etc/hosts
```

### Testing HTTPS
```bash
# Use -k to ignore the self-signed certificate warning
curl -vk https://longhorn.local
curl -vk https://hubble.local
```

> [!WARNING]
> Because we use a **Self-Signed ClusterIssuer**, browsers will show a security warning. You must manually "Proceed" to view the UI.

---

> [!TIP]
> You can now access your cluster dashboards at `https://longhorn.local` and `https://hubble.local` without needing `kubectl port-forward`.
