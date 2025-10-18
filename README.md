# k8s-homelab-gitops

GitOps for my Proxmox homelab (**pve02**). Kubernetes + Argo CD manage core apps and VMs: **nginxLoadbalancer**, **todolist**, **debianMSSQLServer006** (SQL Server), **octopus2025.3.14287** (Octopus Deploy), **ollama.webui** (Ollama/Open WebUI), and a **Windows 10** VM. Traefik, MetalLB, Longhorn, Prometheus & Grafana.

> Repo: https://github.com/angelocarlotto/k8s-homelab-gitops

---

## ⚙️ Prereqs

- Proxmox host (16 cores / 64 GB) running K8s VMs (kubeadm or k3s)
- `kubectl` and `helm`
- DNS for:
  - `argocd.<domain>`, `portainer.<domain>`, `status.<domain>`, `logs.<domain>`
- Traefik as Ingress + MetalLB (or a LoadBalancer provider)
- (Optional) `age`/`sops` if storing secrets in Git

Set commonly used env vars:
```bash
export DOMAIN=home.example.com
export ACME_EMAIL=you@example.com
```

---

## 🗂️ Repo layout

```
.
├─ apps/                   # Helm values / Kustomize bases for components
│  ├─ traefik/
│  ├─ metallb/
│  ├─ longhorn/
│  ├─ portainer/
│  ├─ uptime-kuma/
│  ├─ dozzle/
│  └─ monitoring/          # prometheus, grafana, loki, promtail
├─ clusters/
│  └─ homelab/
│     ├─ apps/             # Argo child Applications (app-of-apps pattern)
│     ├─ networking/       # MetalLB pools, L2Advertisement
│     ├─ storage/          # Longhorn settings / default StorageClass
│     └─ argocd/           # Argo CD ingress, projects, RBAC
├─ external/               # K8s Services pointing to VMs (MSSQL, Win10, etc.)
├─ docs/                   # runbooks, screenshots
└─ README.md
```

---

## 🚀 Bootstrap Argo CD

```bash
kubectl create ns argocd
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n argocd
```

Expose Argo via Traefik:
```yaml
# clusters/homelab/argocd/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls.certresolver: le
spec:
  rules:
    - host: argocd.${DOMAIN}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: argocd-server, port: { number: 80 } }
  tls:
    - hosts: [ "argocd.${DOMAIN}" ]
```
```bash
kubectl apply -f clusters/homelab/argocd/ingress.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## 🧩 App-of-Apps (Argo watches this repo)

```yaml
# clusters/homelab/apps/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homelab
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/angelocarlotto/k8s-homelab-gitops.git
    targetRevision: main
    path: clusters/homelab/apps
    directory: { recurse: true }
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ "CreateNamespace=true" ]
```

Each child app lives under `clusters/homelab/apps/` (Traefik, MetalLB, Longhorn, Portainer, Uptime-Kuma, Dozzle, Monitoring, etc.).

---

## 🌐 Networking

- **MetalLB:** reserve a pool like `192.168.1.240-192.168.1.250`
- **Traefik:** TLS via ACME (Let’s Encrypt) using `${ACME_EMAIL}`
- **External VMs:** point K8s to Proxmox VMs with `ExternalName` or headless `Service`+`Endpoints`:

```yaml
apiVersion: v1
kind: Service
metadata: { name: mssql-external, namespace: infra }
spec:
  type: ExternalName
  externalName: debianMSSQLServer006.lan
```

---

## 💾 Storage

- **Longhorn** provides the default `StorageClass`
- Attach an extra virtual disk to each **worker** VM; Longhorn will use it for replicated volumes
- Backups: set an S3/MinIO or NFS target in Longhorn UI

---

## 🔐 Secrets

Don’t commit raw secrets. Options:
- **SOPS + age** (encrypted in Git)
- **External Secrets Operator** (pull from 1Password/Vault/Bitwarden)
- **Sealed Secrets** (cluster-bound encryption)

Document the chosen approach in `docs/runbooks.md`.

---

## 📦 Apps in this homelab

- **Portainer** (K8s) — cluster UI  
- **Uptime-Kuma** — `status.${DOMAIN}`  
- **Dozzle** — `logs.${DOMAIN}`  
- **Traefik** — Ingress + ACME  
- **Monitoring** — Prometheus, Grafana, Loki/Promtail  
- **External VMs** (documented in `external/`):  
  **nginxLoadbalancer**, **todolist**, **debianMSSQLServer006**, **octopus2025.3.14287**, **ollama.webui**, **windows10**

---

## 🔄 Workflow

1. Edit manifests/values → commit → push to `main`  
2. Argo CD detects changes and **syncs**  
3. Rollback = revert commit (Argo **self-heals**)

---

## 🧭 Roadmap

- Add ApplicationSet for per-env overlays
- Harden TLS & SSO (Authelia/Keycloak)
- GitHub Actions to lint/validate manifests

---

## 📜 License

MIT – see `LICENSE`.
