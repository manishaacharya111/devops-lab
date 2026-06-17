# Week 2 — Ingress, RBAC, Network Policies

## What was built

A complete security and routing layer on top of the week 1 deployments.

## Components

### Ingress
- Installed nginx ingress controller via kind deploy manifest
- Created two deployments: `hello-app` and `hello-app-v2`
- Configured path-based routing: `/v1` → hello-app-service, `/v2` → hello-app-v2-service
- Verified routing end-to-end via port-forward tunnel (kind limitation — no real LB)
- Used `rewrite-target: /` annotation to strip path prefix before forwarding to pods

### RBAC
- Created `pod-reader` ServiceAccount
- Created `pod-reader-role` Role — allows only get/list/watch on pods
- Created RoleBinding connecting the two
- Verified with `kubectl auth can-i` — list pods: yes, delete pods: no, list secrets: no
- Tested live permission changes — take effect instantly with no restarts

### Network Policies
- Applied `default-deny-all` — blocks all ingress and egress for every pod in namespace
- Applied `allow-ingress-to-hello-app` — permits only ingress-nginx namespace to reach hello-app pods on port 80
- Verified with busybox pod:
  - Before policy: immediate HTML response from pod IP
  - After policy: `wget: download timed out` — packets silently dropped

## Key learnings

- Ingress needs both an Ingress object (rules) and Ingress controller (implementation) — one without the other does nothing
- `rewrite-target` strips the path prefix — without it pods return 404 because they don't know about `/v1`
- kind runs inside Docker inside Mac — three network layers mean port-forward is needed to bridge Mac to cluster
- RBAC `podSelector: {}` means ALL pods, not no pods — empty selector = match everything
- Network Policy drops packets silently — connection times out rather than immediately refusing
- Egress deny-all also blocks DNS — always add port 53/UDP egress rule in production

## Files

```
week2/
├── README.md
├── deployment-v2.yaml     # hello-app-v2 deployment + service
├── ingress.yaml           # path-based routing rules
├── rbac.yaml              # ServiceAccount + Role + RoleBinding
└── netpol.yaml            # default-deny-all + allow ingress-nginx
```

## Commands used

```bash
# Install ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Port-forward to test ingress (kind specific)
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80

# Test path routing
curl -H "Host: hello.local" http://localhost:8080/v1
curl -H "Host: hello.local" http://localhost:8080/v2

# Test RBAC permissions
kubectl auth can-i list pods --as=system:serviceaccount:default:pod-reader
kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-reader

# Test network policy blocking
kubectl run test-pod --image=busybox -it --rm -- wget -O- <pod-ip>:80 --timeout=3
```