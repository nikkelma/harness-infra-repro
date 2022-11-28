# Notes - k8s 1.20, Istio 1.9, private PKI / CA chains

## Install cert manager

### Install helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Install cert-manager CRDs and chart

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml

helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.9.1
```

## Create cert-manager resources

> NOTE: if no DNS names are available, use IP in an sslip.io DNS name as a workaround.

```bash
# supports multiple independent CA chains - uncomment b array entry
chain_ids=("a") # "b")

wildcard_base="10.1.2.3.sslip.io"

for id in "${chain_ids[@]}"; do
  echo "\
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ss-chain-${id}-root
spec:
  selfSigned: {}" \
  > "cluster-issuer_ss-chain-${id}-root.yaml"

  kubectl apply -f "cluster-issuer_ss-chain-${id}-root.yaml"

  echo "\
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: chain-${id}-ca-root
  namespace: cert-manager
spec:
  isCA: true
  subject:
    organizations:
      - harness
  commonName: harness-chain-${id}-root
  secretName: chain-${id}-root-tls
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: ss-chain-${id}-root
    kind: ClusterIssuer
    group: cert-manager.io" \
    > "certificate_chain-${id}-ca-root.yaml"

  kubectl apply -f "certificate_chain-${id}-ca-root.yaml"

  echo "\
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: chain-${id}-ca-root
  namespace: cert-manager
spec:
  ca:
    secretName: chain-${id}-root-tls" \
    > "issuer_chain-${id}-ca-root.yaml"

  kubectl apply -f "issuer_chain-${id}-ca-root.yaml"

  echo "\
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: chain-${id}-ca-inter
  namespace: cert-manager
spec:
  isCA: true
  subject:
    organizations:
      - harness
  commonName: harness-chain-${id}-inter
  secretName: chain-${id}-inter-tls
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: chain-${id}-ca-root
    kind: Issuer
    group: cert-manager.io" \
    > "certificate_chain-${id}-ca-inter.yaml"

  kubectl apply -f "certificate_chain-${id}-ca-inter.yaml"

  echo "\
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: chain-${id}-ca-inter
  namespace: cert-manager
spec:
  ca:
    secretName: chain-${id}-inter-tls" \
    > "issuer_chain-${id}-ca-inter.yaml"

  kubectl apply -f "issuer_chain-${id}-ca-inter.yaml"

  echo "\
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: chain-${id}-root-wildcard
  namespace: cert-manager
spec:
  secretName: chain-${id}-root-wildcard-tls
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  subject:
    organizations:
      - harness
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - \"*.${wildcard_base}\"
  issuerRef:
    name: chain-${id}-ca-root
    kind: ClusterIssuer
    group: cert-manager.io" \
    > "certificate_chain-${id}-root-wildcard.yaml"

  kubectl apply -f "certificate_chain-${id}-root-wildcard.yaml"
done
```

## Install gitea

```bash
wildcard_base="10.1.2.3.sslip.io"

echo "\
persistence:
  enabled: true
  size: 10Gi
gitea:
  admin:
    username: gitea_admin
    password: VvPAu0zzOBwBWYrfselr
  config:
    server:
      DOMAIN: gitea.${wildcard_base}
      SSH_DOMAIN: gitea.${wildcard_base}
      ROOT_URL: https://gitea.${wildcard_base}" \
  > values.yaml

helm repo add gitea-charts https://dl.gitea.io/charts/
helm repo update

helm upgrade --install gitea gitea-charts/gitea \
  -n gitea --create-namespace \
  --version 6.0.3 \
  -f values.yaml
```

### Allow network ingress

Create certificate
```bash
echo "\
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: chain-a-root-gitea
  namespace: gitea
spec:
  secretName: chain-a-root-gitea-tls
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  subject:
    organizations:
      - harness
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - gitea.${wildcard_base}
  issuerRef:
    name: chain-a-ca-root
    kind: ClusterIssuer
    group: cert-manager.io" \
    > certificate_chain-a-root-gitea.yaml
  
kubectl apply -f certificate_chain-a-root-gitea.yaml
```

Istio: create VirtualService

> NOTE: change istio gateway to reflect your gateway namespace/name

```bash
echo "\
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: gitea-ui
  namespace: gitea
spec:
  hosts:
  - gitea.${wildcard_base}
  gateways:
  # CHANGE GATEWAY if needed
  - istio-system/harness
  http:
  - name: gitea-ui-base
    match:
    - uri:
        prefix: /
    route:
    - destination:
        host: gitea-http.gitea.svc.cluster.local
        port:
          number: 3000" \
  > virtual-service_gitea-ui.yaml

  kubectl apply -f virtual-service_gitea-ui.yaml
```

Ingress: create Ingress

NOTE: fill in ingressClassName if needed

```bash
echo "\
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ui
  namespace: gitea
spec:
  # ingressClassName: nginx
  rules:
    - host: gitea.${wildcard_base}
      http:
        paths:
          - backend:
              service:
                name: gitea-http
                port:
                  number: 3000
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - gitea.${wildcard_base}
      secretName: chain-a-root-gitea-tls" \
  > ingress_gitea-ui.yaml

kubectl apply -f ingress_gitea-ui.yaml
```

