# OCI k3s + GitOps — Architecture, Steps & Discussion

> **Status:** Phase 1 complete — single-node k3s is installed and verified on the data volume. Phase 0 inputs and Phase 2 (Argo CD bootstrap) are next. No application workloads have been deployed yet. The Homepage dashboard and wildcard subdomain routing (§4.8–§4.10) were added to the design on 2026-10-21; none of it is deployed yet.
> **Target:** Small single-node ARM VM on Oracle Cloud (resource class intentionally generalized).
> **Goal:** Run n8n, Nextcloud, and related open-source apps with the k3s + Argo CD GitOps architecture from the referenced article — not Docker Compose. `https://<DOMAIN>` opens a Homepage dashboard with one card per app; each card links to the app's own subdomain, such as `https://n8n.<DOMAIN>`.
>
> **Privacy note:** No public IPs, hostnames, domains, usernames, local paths, or credentials are recorded in this document. All environment-specific values are placeholders.

---

## 1. Executive summary

The plan is to build a single-node k3s cluster on the existing OCI ARM VM, manage workloads with Argo CD using an app-of-apps pattern, expose selected apps through a Cloudflare Tunnel, and keep secrets out of Git using Infisical + External Secrets Operator (ESO).

Recommended shape:

- **One domain**, DNS managed by Cloudflare.
- **Cloudflare Tunnel** for public access; no inbound `80`/`443` and no A record to the VM.
- **Two tunnel hostnames only:** `<DOMAIN>` and `*.<DOMAIN>`, both routed to Traefik. Adding an app takes one Kubernetes `Ingress`; DNS and the tunnel stay unchanged.
- **Homepage dashboard** at `<DOMAIN>`. Cards are discovered from app `Ingress` annotations and link to `<app>.<DOMAIN>`.
- **No load balancer:** no MetalLB, k3s ServiceLB, or OCI Load Balancer. `cloudflared` connects outbound and reaches Traefik through its `ClusterIP` Service.
- **Cloudflare Access** gates `<DOMAIN>` and `*.<DOMAIN>` before any hostname goes live.
- **Tailscale** for private admin access (SSH, kubectl, Argo CD) where possible.
- **k3s** with Traefik and ServiceLB disabled; Traefik installed separately as a `ClusterIP` service.
- **Argo CD** bootstrapped once, then managing itself and all apps from Git.
- **Infisical Cloud free tier** for phase 1; self-host later only if needed.
- **Backups:** OCI volume backups **plus** application-consistent dumps to OCI Object Storage via restic.
- **Phased rollout:** core first; Zot, self-hosted GitHub runner, AdGuard, and Headlamp deferred.

The full article stack is tight but plausible on a small single-node VM if optional components are deferred and resource requests are kept modest.

---

## 2. Target environment and assumptions

| Item | Assumption |
|---|---|
| Host | Single-node Linux VM on Oracle Cloud |
| Architecture | ARM64 |
| Resources | Small single-node class; exact host details intentionally omitted |
| Storage | Root disk plus a verified ext4 data volume mounted at `<DATA_VOLUME_MOUNT>`; k3s state, container images, and app PVCs live there |
| Container runtime | k3s-managed containerd; no assumption about preinstalled Docker |
| Inbound network | SSH only by design; public app traffic goes through a Tunnel |
| Repository | Existing files are unrelated to the target stack; local environment files must never be committed |

No public IP, hostname, username, disk identifier, or credential is recorded here.

The data volume is mounted persistently via `/etc/fstab` using a UUID with `_netdev,nofail`. Because `nofail` alone can let k3s start with an empty root-disk directory, the mount guard in §7.1 is installed and active.

---

## 3. Target architecture

### 3.1 Diagram

The editable Excalidraw source is [`OCI-K3S-ARCHITECTURE.excalidraw`](OCI-K3S-ARCHITECTURE.excalidraw), with a rendered [`SVG`](OCI-K3S-ARCHITECTURE.svg). No public share link is embedded. Updated 2026-10-21 for the Access-first setup, wildcard routing to the Homepage dashboard and apps (§4.8–§4.10), and Renovate image updates (§6.7).

It covers five lanes:

1. One-time domain setup
2. Public request path
3. Private admin path
4. GitOps, secrets, and image updates
5. Backups

### 3.2 Component map

| Layer | Component | Purpose | Phase |
|---|---|---|---|
| OS | Linux + k3s | Single-node Kubernetes | 1 |
| GitOps | Argo CD | App-of-apps, sync from Git | 2 |
| Ingress | Traefik v3 | Host-based routing inside cluster | 3 |
| Tunnel | `cloudflared` | Outbound-only public ingress | 3 |
| TLS | Cloudflare edge | Public HTTPS termination | 3 |
| Dashboard | Homepage | Landing page at `<DOMAIN>`; app cards from `Ingress` annotations | 3 |
| Optional TLS | cert-manager + Cloudflare DNS-01 | Internal/private TLS or future direct ingress | 3+ |
| Secrets | Infisical + ESO | Secret source of truth → Kubernetes Secrets | 4 |
| Apps | n8n, Nextcloud | Primary workloads | 5 |
| Auth | Cloudflare Access, optional Authelia | Edge gate (Access) and app-level SSO (Authelia) | 3, 6 |
| Backups | OCI volume backups + restic + `pg_dump` | Disk DR + app-consistent backups | 7 |
| Deferred | Zot, GitHub runner, AdGuard, Headlamp | Registry, CI, DNS, UI | 8 |

### 3.3 Public request path

```text
User
 │  https://<DOMAIN>  or  https://n8n.<DOMAIN>
 ▼
Cloudflare edge
 │  TLS terminated at Cloudflare (Universal SSL: <DOMAIN> + *.<DOMAIN>)
 │  Cloudflare Access login
 ▼
Encrypted Cloudflare Tunnel
 │  cloudflared in k3s connects OUT to Cloudflare
 │  tunnel publishes only <DOMAIN> and *.<DOMAIN>, both → Traefik
 ▼
Traefik (ClusterIP)
 │  routes by Host header
 ├─ <DOMAIN>        → Homepage (dashboard cards)
 ├─ n8n.<DOMAIN>    → n8n Service → n8n Pod
 └─ cloud.<DOMAIN>  → Nextcloud Service → Nextcloud Pod
```

No inbound `80`/`443`. No A record to the VM. No port forwarding. No load balancer (§4.8).

### 3.4 Private admin path

```text
Admin device
 │  Tailscale
 ▼
VM Tailscale IP <TAILSCALE_IP>
 │  SSH / kubectl port-forward
 ▼
Argo CD, Kubernetes API, Grafana, etc.
```

Use this for admin surfaces where public exposure is unnecessary.

---

## 4. Domain and access flow

### 4.1 Short answer

You need **one domain**, but you do **not** point it at the VM's public IP.

Your current registrar is fine. Move the domain's nameservers to Cloudflare, then route hostnames through a Cloudflare Tunnel.

The tunnel publishes exactly two hostnames: `<DOMAIN>` (the Homepage dashboard) and `*.<DOMAIN>` (every app). Traefik picks the app from the `Host` header, so a new app needs only an `Ingress`; DNS and the tunnel stay unchanged.

### 4.2 Exact steps

1. **Buy or choose one domain.** One apex domain is enough; use subdomains per app.
2. **Add the domain to Cloudflare** (Free plan).
3. Cloudflare assigns two nameservers, for example:
   - `<NS_1>`
   - `<NS_2>`
4. **At the registrar**, replace the existing nameservers with Cloudflare's pair. Do not create an A record to the VM.
5. Wait until Cloudflare shows the zone as **Active**.
6. **Create a Tunnel** in Cloudflare Zero Trust → Networks → Tunnels. Use the token-based install; copy the tunnel token.
7. **Deploy `cloudflared` in k3s** with the token stored as a Kubernetes Secret.
8. **Create the Cloudflare Access gate first** (§4.6). With a wildcard hostname, every `Ingress` is public as soon as it exists, and Homepage has no built-in authentication.
9. **Add two Public Hostnames** in the tunnel, both → `http://traefik.kube-system.svc.cluster.local:80`:
   - `<DOMAIN>` — the dashboard
   - `*.<DOMAIN>` — every app

   The same ingress rules through the Cloudflare API (`PUT /accounts/<ACCOUNT_ID>/cfd_tunnel/<TUNNEL_ID>/configurations`), as the JobLens AI deploy script does:

   ```json
   "ingress": [
     {"hostname": "<DOMAIN>",   "service": "http://traefik.kube-system.svc.cluster.local:80"},
     {"hostname": "*.<DOMAIN>", "service": "http://traefik.kube-system.svc.cluster.local:80"},
     {"service": "http_status:404"}
   ]
   ```

10. **DNS:** Cloudflare creates the apex **proxied CNAME** to `<TUNNEL_ID>.cfargotunnel.com`. If the dashboard does not create the wildcard record, add a proxied CNAME `*` → `<TUNNEL_ID>.cfargotunnel.com` manually.
11. **OCI firewall:** allow SSH `22` only from the admin source; allow outbound `7844` TCP/UDP; close inbound `80`, `443`, `6443`.
12. **TLS:** Cloudflare's edge certificate handles public HTTPS. The `cloudflared → Traefik` hop stays inside the encrypted tunnel, so plain HTTP is acceptable. Turn on **SSL/TLS → Edge Certificates → Always Use HTTPS**.
13. **Authelia (later):** avoid stacking it and Cloudflare Access on the same hostname.

### 4.3 DNS records

| Hostname | Type | Target | Proxy |
|---|---|---|---|
| `<DOMAIN>` (`@`) | CNAME | `<TUNNEL_ID>.cfargotunnel.com` | Proxied |
| `*.<DOMAIN>` (`*`) | CNAME | `<TUNNEL_ID>.cfargotunnel.com` | Proxied |

No A record to the VM, and no per-app records: `n8n.<DOMAIN>`, `cloud.<DOMAIN>`, `auth.<DOMAIN>`, and `argocd.<DOMAIN>` all resolve through the wildcard. A specific record, if one is ever added, overrides the wildcard for that name. Cloudflare allows proxied wildcard records on all plans.

### 4.4 Firewall rules

| Direction | Port | Source/Destination | Purpose |
|---|---|---|---|
| Ingress | `22/tcp` | Admin source only | SSH |
| Ingress | `80`, `443`, `6443` | Deny | Not needed with Tunnel |
| Egress | `7844/tcp+udp` | Cloudflare edge | Tunnel (`http2`/`quic`) |
| Egress | `443/tcp` | Cloudflare, Infisical, OCI Object Storage, GitHub, registries | APIs and updates |
| Egress | `53/tcp+udp` | DNS resolver | Name resolution |

Cloudflare documents `7844` TCP/UDP as the required tunnel egress port; `443` is optional for updates and some features.

### 4.5 TLS and cert-manager

With Cloudflare Tunnel, public TLS is handled at Cloudflare's edge. A cert-manager wildcard certificate via Cloudflare DNS-01 is **not required for the public path**.

Cloudflare's free Universal SSL certificate covers `<DOMAIN>` and first-level subdomains only. Keep app hostnames one level deep: `n8n.<DOMAIN>` works; `n8n.lab.<DOMAIN>` gets no valid edge certificate.

Keep cert-manager only if:

- you want internal/private hostnames with TLS,
- you later switch to direct ingress,
- or you want end-to-end TLS from `cloudflared` to Traefik.

This is a deliberate simplification from the article. The article's cert-manager + Traefik wildcard is still valid, but it is not needed just to get public HTTPS through the Tunnel.

### 4.6 Cloudflare Access vs Authelia

Do not stack both on the same hostname or you will get a double login.

**Baseline gate (Phase 3):** one Cloudflare Access self-hosted application covering `<DOMAIN>` and `*.<DOMAIN>`, with an Allow policy for the admin email(s). The Free plan covers up to 50 users. Add separate Bypass applications for paths that must stay public, such as n8n webhooks and Nextcloud shares (§4.7).

Options:

- **Simplest:** Cloudflare Access only for admin apps; skip Authelia initially.
- **Article-style:** Authelia as the app SSO gate; Cloudflare Access only where needed.
- **Hybrid:** Cloudflare Access for infra/admin (Argo CD, Headlamp, Grafana), Authelia for end-user apps (n8n, Nextcloud).

If Authelia is used as an OIDC provider, `auth.<DOMAIN>` (or the OIDC endpoints) needs a Cloudflare Access bypass so the login redirect can complete.

### 4.7 Important gotchas

- **n8n webhooks:** if Cloudflare Access protects `n8n.<DOMAIN>`, external webhook calls get redirected to a login page and fail. Bypass `/webhook/*`, use a separate `hooks.<DOMAIN>` hostname, or use a Cloudflare service token.
- **Nextcloud public shares:** same issue; share links need an Access bypass or a separate hostname.
- **Cloudflare Tunnel token mode:** hostnames live in the Cloudflare dashboard, not Git. With the wildcard, only `<DOMAIN>` and `*.<DOMAIN>` live there; per-app hostnames live in Git as `Ingress` objects. If you want tunnel config in Git, use a locally-managed tunnel later.
- **OCI ephemeral public IP:** can change on stop/start. Another reason not to hardcode an A record. Reserve the IP only if you ever use direct ingress.
- **Zone must be on Cloudflare:** named Tunnel hostnames require the DNS zone to be managed by Cloudflare. Registrar can stay elsewhere.
- **Wildcard does not cover the apex:** `*.<DOMAIN>` matches subdomains only, so `<DOMAIN>` needs its own tunnel hostname and DNS record.
- **Cloudflare Free limits:** requests larger than 100 MB are rejected with `413`, and Cloudflare's CDN terms restrict serving video and large files that are not hosted on Cloudflare (Stream, R2). Reach media apps (Jellyfin, Immich) and very large uploads over Tailscale or the direct-IP alternative (§4.12).
- **Scheme behind the tunnel:** Traefik must trust `X-Forwarded-*` headers from the `cloudflared` pods (§7.3); otherwise apps see `http` and can loop on HTTPS redirects.

### 4.8 No load balancer

No MetalLB, no k3s ServiceLB, and no OCI Load Balancer. Nothing arrives from the internet: `cloudflared` dials out to Cloudflare and reaches Traefik through its `ClusterIP` Service, so there is no inbound traffic to balance.

| Option | Verdict | Reason |
|---|---|---|
| MetalLB | No | Built for bare metal; its docs say it is generally not compatible with cloud providers, and OCI is not on its list. Layer 2 mode relies on ARP announcements that cloud networks generally do not honor; Oracle's MetalLB lab requires enabling Layer 2 networking for VLANs in the VCN, which it describes as not generally available. The VM has one public IP anyway. |
| k3s ServiceLB | No (stays disabled) | Only needed if Traefik must bind `80`/`443` on the node — the direct-IP alternative (§4.12). |
| OCI Load Balancer | No | The Always Free Flexible Load Balancer is fixed at 10 Mbps; on a single node it adds a hop and gains nothing. |
| Tunnel → Traefik `ClusterIP` | **Yes** | Free, no open ports, origin IP hidden, no reserved IP needed. |

**JobLens AI comparison.** That project's local cluster (`infra/local-k8s`) runs KinD, MetalLB, and an nginx `api-gateway` Service of `type: LoadBalancer`. MetalLB exists there only because KinD on Docker Desktop has no LoadBalancer provider and the Mac needs a reachable Docker-bridge IP. None of that applies to an OCI VM.

| Reuse from JobLens AI | Drop |
|---|---|
| `cloudflared` Deployment (token Secret, `/ready` probes); bump its pinned `2025.1.0` image to a current release (`2026.10.2` as of 2026-10-21) | KinD and MetalLB |
| Tunnel configuration through the Cloudflare API (`PUT .../cfd_tunnel/<TUNNEL_ID>/configurations`) | nginx `api-gateway` `LoadBalancer`; Traefik `Ingress` is the gateway here |
| Argo CD `Application` objects with automated prune and self-heal | Gitea (use the private GitOps repo) and Vault (this plan uses Infisical, or SOPS) |

### 4.9 Dashboard at `<DOMAIN>` (Homepage)

[Homepage](https://gethomepage.dev) serves the landing page. With `mode: cluster` it reads the Kubernetes API through its ServiceAccount and builds one card per annotated `Ingress`, linking to that app's subdomain. No card list is maintained by hand.

Deploy it from the official raw manifests on Homepage's Kubernetes install page (ServiceAccount, token Secret, ConfigMap, ClusterRole and binding, Service, Deployment, Ingress), synced by Argo CD from `manifests/homepage/`. Skip the unofficial Helm chart: its latest release (May 2025) ships Homepage v1.2.0, while Homepage is at v2.6.0 (released 2026-10-17).

Changes from the docs example:

```yaml
# ConfigMap: kubernetes.yaml
mode: cluster              # in-cluster ServiceAccount; Ingress discovery is on by default

# Deployment
image: ghcr.io/gethomepage/homepage:v2.6.0   # pinned; the docs example uses :latest
env:
  - name: MY_POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: HOMEPAGE_ALLOWED_HOSTS
    value: "$(MY_POD_IP):3000,<DOMAIN>"      # pod IP entry is needed for the probes; never "*"

# Ingress
spec:
  ingressClassName: traefik
  rules:
    - host: <DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homepage
                port:
                  number: 3000
```

- Homepage has **no built-in authentication**; its docs require an authenticating reverse proxy for public deployments. The Access gate (§4.6) must exist before `<DOMAIN>` goes live.
- Cluster CPU/memory widgets use metrics-server, which k3s already runs.
- Service widgets call app APIs from the Homepage pod: point `gethomepage.dev/widget.url` at the internal Service URL (`http://<svc>.<namespace>.svc.cluster.local:<port>`), not the public hostname, which Access would block.

### 4.10 Adding an app

Each app needs a `ClusterIP` Service and one standard `Ingress`. The wildcard hostname and DNS record already cover the subdomain, and Homepage adds the card on its own.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: n8n
  namespace: n8n
  annotations:
    gethomepage.dev/enabled: "true"
    gethomepage.dev/name: n8n
    gethomepage.dev/description: Workflow automation
    gethomepage.dev/group: Automation
    gethomepage.dev/icon: n8n.png
    gethomepage.dev/href: https://n8n.<DOMAIN>
    gethomepage.dev/pod-selector: app.kubernetes.io/name=n8n
spec:
  ingressClassName: traefik
  rules:
    - host: n8n.<DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: n8n
                port:
                  number: 5678
```

- **Always set `gethomepage.dev/href`.** Homepage v2.6.0 uses `https` in a card link only when the `Ingress` has a `tls:` block (`src/utils/kubernetes/resource-helpers.js`). Tunnel-routed Ingresses need no `tls:` block because TLS ends at Cloudflare's edge, so without the annotation the card links to `http://`.
- **Use `Ingress`, not Traefik `IngressRoute`,** for app routes. Homepage discovers `IngressRoute` objects only with `traefik: true` in `kubernetes.yaml`, and ignores any that lack `gethomepage.dev/href`.
- `gethomepage.dev/pod-selector` selects the pods behind the card's status.

### 4.11 Private-only alternative

If you do not want a public domain:

- Install Tailscale on the VM and your admin device.
- Access n8n at `http://<TAILSCALE_IP>:5678`, or use Tailscale Serve for HTTPS.
- Use `kubectl port-forward` over SSH for Argo CD.
- No domain, no public exposure.

Trade-off: n8n external webhooks and Nextcloud public shares will not work without Tailscale Funnel or a public tunnel.

**Recommendation:** Cloudflare Tunnel + domain for public apps; Tailscale for SSH/kubectl/Argo CD admin.

### 4.12 Direct public IP alternative

Use only if Cloudflare's limits (§4.7) block a workload. It still needs no MetalLB:

1. Remove `--disable servicelb` from the k3s server flags and set Traefik's Service back to `LoadBalancer`; ServiceLB then binds `80`/`443` on the node.
2. Open `80`/`443` in the OCI security list **and** the host firewall; the Oracle Ubuntu image rejects all new inbound connections except SSH.
3. Reserve the public IP so DNS survives instance recreation.
4. DNS: `A @` and `A *` → the public IP.
5. Issue a cert-manager wildcard certificate for `<DOMAIN>` and `*.<DOMAIN>` via Cloudflare DNS-01 and set it as Traefik's default certificate.

Trade-offs: the origin IP is exposed and scanned, certificate renewal becomes your job, and Cloudflare Access cannot gate DNS-only records. Not the default.

---

## 5. Secrets with Infisical

### 5.1 What it is

Infisical is an open-source secrets manager with a UI, API, CLI, versioning, audit logs, access control, and machine identities. It is the source of truth for application secrets — not a personal password manager.

Examples of secrets:

- database passwords
- API keys
- OAuth client secrets
- n8n encryption key
- Nextcloud admin password
- SMTP credentials
- Authelia secrets

### 5.2 Flow

```text
You edit a secret in Infisical
        │
        ▼
External Secrets Operator (ESO) in k3s
        │  authenticates as a machine identity
        ▼
Creates/updates a normal Kubernetes Secret
        │
        ▼
n8n / Nextcloud / Authelia pods consume it
```

The Git repo contains an `ExternalSecret` object that references the secret. It does not contain the value.

### 5.3 Why use it

- No plaintext environment files or secrets in Git.
- One place to rotate database passwords and API keys.
- Audit trail of reads and changes.
- One secret can feed multiple apps.
- ESO refreshes secrets and can restart workloads.

### 5.4 Bootstrap secrets (chicken-and-egg)

These cannot come from Infisical and must be created manually or with SOPS/Sealed Secrets before Argo CD syncs:

- ESO's Infisical machine identity credential
- Infisical's own encryption key (if self-hosted)
- Cloudflare Tunnel token
- Argo CD admin password
- k3s cluster token
- restic backup password
- OCI API credentials for backups

Argo CD can manage `ExternalSecret` objects, but ESO's auth Secret must exist first.

### 5.5 Cloud vs self-hosted

| Option | Pros | Cons |
|---|---|---|
| Infisical Cloud free tier | No Postgres/Redis to run; quick; free plan includes 5 identities at time of writing | External dependency; free-tier limits |
| Self-hosted Infisical | Data stays on the VM; full control | Adds Infisical + Postgres + Redis; more RAM, backup, and upgrade work |

**Recommendation:** Infisical Cloud free tier for phase 1. Move self-hosted later if data sovereignty or feature needs justify it.

ESO has a native Infisical provider. Universal Auth (client ID + client secret) is the simplest setup. Kubernetes Auth is possible but needs TokenReview permissions. Infisical also has its own Kubernetes Operator; do not run both ESO and the Infisical Operator for the same secrets.

### 5.6 Alternative

For a solo environment, **SOPS + age** is simpler than Infisical: secrets encrypted in Git, no extra service. Infisical is justified if you want the UI, rotation, audit, and central management. The article uses Infisical, so keeping it is reasonable.

---

## 6. GitOps with Argo CD

### 6.1 Key clarification

Argo CD runs **inside** Kubernetes. It does **not** contain your application manifests. It watches Git and applies what it finds.

There are two layers.

### 6.2 Layer 1 — Bootstrap

Argo CD must be installed before it can manage anything. This is the chicken-and-egg step:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/<PINNED_VERSION>/manifests/install.yaml
# or a pinned Helm chart
```

Then apply the root Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <GITOPS_REPO_URL>
    path: apps
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This bootstrap step is outside Argo CD. Argo CD cannot install itself before it exists.

### 6.3 Layer 2 — App-of-apps

The root app points to `apps/`, which contains child `Application` objects:

```text
gitops/
├── bootstrap/
│   └── argocd/                 # applied once with kubectl/Helm
├── root-app.yaml               # applied once
├── apps/
│   ├── traefik.yaml
│   ├── cert-manager.yaml
│   ├── external-secrets.yaml
│   ├── infisical.yaml
│   ├── cloudflared.yaml
│   ├── authelia.yaml
│   ├── homepage.yaml
│   ├── n8n.yaml
│   └── nextcloud.yaml
└── manifests/
    ├── homepage/
    ├── n8n/
    └── nextcloud/
```

Each child app can point to:

- a Helm chart + values file,
- a Kustomize directory,
- or plain YAML in `manifests/`.

### 6.4 What lives where

| Item | Location |
|---|---|
| Argo CD install manifests/Helm values | `bootstrap/argocd/` |
| Root Application | `root-app.yaml` |
| Child Applications | `apps/` |
| App manifests / Helm values | `manifests/` |
| Plaintext secrets | **Never in Git** |
| `ExternalSecret` objects | Git |
| Infisical machine credential | Bootstrap Secret, not Git |
| Cloudflare tunnel token | Bootstrap Secret, not Git |

### 6.5 Self-management

Argo CD can manage itself after bootstrap by including its own Helm chart in the root app. But if Argo CD is broken, it cannot heal itself — you re-bootstrap it. Keep the bootstrap manifests and instructions in Git so recovery is repeatable.

### 6.6 Sync order

Install CRDs and controllers before custom resources:

1. Namespaces
2. Traefik, cert-manager, ESO, Infisical
3. `SecretStore` / `ExternalSecret`
4. Homepage, n8n, Nextcloud
5. Authelia, cloudflared

Use `argocd.argoproj.io/sync-wave` annotations or separate apps.

### 6.7 Image updates

Argo CD deploys what Git says. It polls the GitOps repo every three minutes; it does **not** watch container registries. A new upstream n8n image changes nothing until its tag changes in Git.

Pin every image tag and chart version in Git, and let a bot bump them:

```text
n8n publishes a new image
        │
        ▼
Renovate opens a PR bumping the tag in manifests/n8n/kustomization.yaml
        │  review it, or automerge patch releases
        ▼
Merge to main
        │
        ▼
Argo CD syncs within ~3 minutes → rolling update
```

| Option | Verdict | Notes |
|---|---|---|
| Renovate (free GitHub app) | **Recommended** | Runs outside the cluster, so it costs the VM nothing. Every upgrade is a reviewable, revertable PR. Also bumps Helm chart versions (Traefik, Argo CD) and images such as `cloudflared` and Homepage. The `kustomize` manager reads `images:` in `kustomization.yaml` by default; the `argocd` and `kubernetes` managers need `managerFilePatterns`. |
| Argo CD Image Updater (`v1.3.1` as of 2026-10-21) | Alternative | In-cluster controller that polls registries and writes the new tag back to Git. Configured with the `ImageUpdater` CRD since v1.0. Supports Helm and Kustomize apps, not plain manifests. Adds a controller and registry polling to a small VM. |
| `:latest` + `imagePullPolicy: Always` | Avoid | Git never changes, so Argo CD never redeploys; upgrades land at random pod restarts, with no audit trail or rollback. |

Rules:

- Keep each app's image in a Kustomize `images:` entry (or a Helm `image.tag` value) so either tool can update it.
- Automerge only patch updates of stateless apps. n8n and Nextcloud run database migrations on upgrade: take the Phase 7 dump, then merge.
- A GitHub webhook to Argo CD removes the three-minute delay, but it has to pass the Access gate (a bypass for the webhook path). Polling is enough.

---

## 7. Storage and k3s layout

### 7.1 k3s data directory

The VM has a separate ext4 block volume mounted at `<DATA_VOLUME_MOUNT>` and recorded in `/etc/fstab` by UUID. Verify it before installing:

```bash
findmnt <DATA_VOLUME_MOUNT>
```

Create a root-owned k3s directory:

```bash
sudo install -d -m 700 -o root -g root <DATA_VOLUME_MOUNT>/k3s
```

Install k3s with the data volume as its data-dir:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --data-dir <DATA_VOLUME_MOUNT>/k3s \
  --disable traefik \
  --disable servicelb \
  --write-kubeconfig-mode 600 \
  --kubelet-arg=container-log-max-size=50Mi \
  --kubelet-arg=container-log-max-files=3" sh -
```

Why:

- k3s server state, containerd images/snapshots, and local-path PVs all live under `<DATA_VOLUME_MOUNT>/k3s`.
- Local-path PVs live under `<DATA_VOLUME_MOUNT>/k3s/storage`.
- Avoids filling the root disk.
- Keeps cluster state on the larger volume.

**Verified in this environment:** k3s `v1.36.5+k3s1` is running with the data volume as its data-dir; the node is `Ready`; CoreDNS, local-path-provisioner, and metrics-server are healthy. A test PVC wrote and read a file under `<DATA_VOLUME_MOUNT>/k3s/storage`, then the PVC/PV/pod were removed cleanly. The mount guard is installed and active; the negative test of starting with the volume unmounted was not performed.

> **`kubectl` note:** `/usr/local/bin/kubectl` is a symlink to the k3s multicall binary. Running bare `sudo kubectl` uses k3s's default data-dir and recreates a second runtime-assets directory on the root disk (observed at roughly 237 MB). Use `sudo k3s --data-dir <DATA_VOLUME_MOUNT>/k3s kubectl ...` or `sudo K3S_DATA_DIR=<DATA_VOLUME_MOUNT>/k3s kubectl ...` instead. The server's systemd unit already uses the data volume.

Guard against a missing mount. If the data volume is not mounted, k3s must not silently create empty state on the root disk:

```bash
sudo systemctl edit k3s.service
```

```ini
[Unit]
RequiresMountsFor=<DATA_VOLUME_MOUNT>

[Service]
ExecStartPre=/usr/bin/mountpoint -q <DATA_VOLUME_MOUNT>
```

Container logs (`/var/log/pods`, `/var/log/containers`) remain on the root disk. The kubelet flags above cap them. Do not move OS directories onto the data volume.

The data volume is not a backup. It is the same single point of failure as the VM; keep OCI volume backups and off-volume restic backups.

### 7.2 Do not merge the boot and data volumes

Keep the boot volume and the data volume separate. Do not concatenate them with LVM, RAID 0, or a union filesystem, and do not try to extend `/` onto the data volume.

Reasons:

- The boot volume is a plain ext4 partition, not LVM. Merging requires repartitioning or reinstalling the OS, which is unnecessary downtime and risk.
- LVM concatenation and RAID 0 do not add redundancy. If either disk fails, the combined volume is damaged or lost.
- RAID 1 would not merge capacity: the usable size would be limited by the smaller boot disk, not the sum.
- The boot volume carries the kernel, bootloader, and EFI partitions. Making it part of a multi-disk data pool complicates boot, recovery, and OCI volume backups.
- k3s does not need the boot disk's free space. Its binary and config are small; state, images, and PVCs belong on the data volume via `--data-dir`.
- Separate volumes let you resize, snapshot, and restore the data volume independently.

If you later want more flexible data-volume management, put LVM on the data disk only. Do not span the boot disk.

### 7.3 Traefik

- Disable k3s's bundled Traefik.
- Install Traefik v3 separately as a `ClusterIP` service (pinned Helm chart; `v42.1.0` as of 2026-10-21).
- Cloudflare Tunnel points at `traefik.kube-system.svc.cluster.local:80`.
- Do not expose Traefik via NodePort or LoadBalancer.
- Trust forwarded headers from the pod network. Requests from `cloudflared` carry `X-Forwarded-Proto: https` from Cloudflare's edge; from an untrusted source, Traefik overwrites it with `http`, and apps then build `http` links or loop on redirects.

```yaml
# Traefik chart values (v42.x keeps the Service type under service.spec)
service:
  spec:
    type: ClusterIP
ports:
  web:
    forwardedHeaders:
      trustedIPs: ["10.42.0.0/16"]   # k3s default cluster CIDR, where cloudflared pods run
```

### 7.4 Swap

- k3s does not tolerate swap by default.
- If swap is enabled later, configure k3s with `--kubelet-arg=fail-swap-on=false` and test carefully.

### 7.5 Single-node persistence

- `local-path` PVs are node-local and not replicated.
- They survive pod restarts but not volume loss or accidental deletion.
- Backups are still required.

---

## 8. Backups

### 8.1 Persistent is not backed up

A Kubernetes PersistentVolume survives a pod restart. It does not survive:

- volume deletion
- disk corruption
- a bad migration
- ransomware
- accidental `kubectl delete`

GitOps backs up manifests, not data.

### 8.2 What OCI backups actually do

OCI Block Volume backup policies are optional. You must assign a policy to the boot volume and the data volume.

They are:

- crash-consistent snapshots of the volume
- cross-volume crash-consistent when using volume groups
- stored in OCI Object Storage, regional
- optionally copied cross-region
- **only data on disk** — application-consistent backups require pausing/quiescing the app

A snapshot of a running Postgres or Nextcloud can capture the database mid-write. It may restore, but the DB may need crash recovery, and files may not match database rows.

They also:

- expire according to retention
- are not granular (single file/workflow restore is painful)
- do not protect against a corrupted database that gets snapshotted
- do not protect against an attacker who can delete both volume and backups
- do not back up anything without a policy assigned

### 8.3 What GitOps does not back up

- n8n workflows/credentials database
- Nextcloud files and database
- Postgres databases
- Infisical database and encryption key
- Authelia user database
- k3s SQLite datastore and cluster token
- Cloudflare tunnel token, restic password, OCI API keys

### 8.4 Recommended layers

**Layer 1 — OCI volume backups**

Assign a policy to the boot volume and the data volume. Use volume group backups. This is fast whole-disk disaster recovery.

**Layer 2 — application-consistent backups**

Run `pg_dump` / `mariadb-dump` and file backups to OCI Object Storage (or another object store) using restic. Stage temporary dumps under `<DATA_VOLUME_MOUNT>/backup-staging` and delete them after upload. Never leave the only copy on the same volume.

| Data | Method | Frequency | Retention |
|---|---|---|---|
| Nextcloud files + DB | maintenance mode → DB dump → files/config → disable maintenance | Daily | 7 daily / 4 weekly / 12 monthly |
| n8n DB + encryption key | `pg_dump` + `N8N_ENCRYPTION_KEY` | Daily | 7 / 4 / 12 |
| Infisical self-hosted | Postgres dump + `ENCRYPTION_KEY` | Daily | 7 / 4 / 12 |
| Authelia | config + user DB + encryption key | Daily | 7 / 4 / 12 |
| k3s | SQLite DB + token + data directory | Daily | 7 / 4 / 12 |
| OCI volumes | Volume backup policy | Daily | Per policy |

**Layer 3 — restore tests**

A backup you have never restored is a hypothesis. Test one app restore monthly on a scratch VM or namespace.

### 8.5 Practical schedule

- Daily: app dumps + restic to Object Storage.
- Daily: OCI volume backup.
- Retain 7 daily / 4 weekly / 12 monthly.
- Quarterly: full restore drill.
- Use a separate OCI compartment/bucket and credentials for backups, so a compromised VM cannot delete them.

### 8.6 Bottom line

Keep OCI volume backups, but add app-level dumps.

- OCI snapshots = "the disk died" recovery.
- `pg_dump` + restic = "the data is wrong" recovery.

You need both.

---

## 9. Phased implementation plan

### Phase 0 — Decisions and prerequisites (in progress)

- [x] Confirm phase 1 scope.
- [x] Preserve existing local configuration; ensure local environment files are gitignored.
- [ ] Confirm domain and Cloudflare nameserver move.
- [ ] Confirm Infisical Cloud vs self-hosted.
- [ ] Create a private GitOps repo.
- [ ] Confirm OCI Object Storage bucket/region for restic.
- [ ] Confirm Authelia timing.

**Done when:** all open decisions are resolved and repo/bucket exist.

### Phase 1 — Base OS and k3s (k3s baseline complete)

- [x] Verify `<DATA_VOLUME_MOUNT>` is mounted; create `<DATA_VOLUME_MOUNT>/k3s` root-owned.
- [x] Install k3s with `<DATA_VOLUME_MOUNT>/k3s` data-dir, Traefik and ServiceLB disabled.
- [x] Set kubeconfig mode `600` and cap container logs at `50Mi` / 3 files.
- [x] Add the systemd mount guard from §7.1.
- [x] Configure the host/OCI firewall so only SSH is externally reachable; `6443` and `10250` are blocked.
- [x] Verify the node is `Ready` and system pods are healthy.
- [x] Create a test PVC and confirm its host path is under `<DATA_VOLUME_MOUNT>/k3s/storage`.
- [ ] Full OS package refresh (not required for the k3s baseline; do before app deployment).

**Done when:** single-node cluster is healthy, a test PVC persists on the data volume, and the mount guard is installed. The negative unmount test was not performed.

### Phase 2 — Argo CD bootstrap

- Install Argo CD (pinned version; `v3.5.6` as of 2026-10-21) into `argocd`.
- Install the `argocd` CLI locally.
- Create the private GitOps repo and root app.
- Reach Argo CD with `kubectl port-forward` or Tailscale; publish it through the Tunnel only after the Phase 3 Access gate exists.
- Apply root app; confirm `Synced`/`Healthy`.
- Pin image tags and chart versions; enable Renovate on the GitOps repo (§6.7).

**Done when:** Argo CD can sync a test app from Git.

### Phase 3 — Ingress, Tunnel, Access gate, and dashboard

- Install Traefik v3 as `ClusterIP` with the forwarded-header trust from §7.3.
- Create the Cloudflare Access application for `<DOMAIN>` and `*.<DOMAIN>` (§4.6) **before** publishing any hostname.
- Create `cloudflared` namespace and token Secret.
- Deploy `cloudflared` (the JobLens AI Deployment with a current pinned image, or the official chart).
- Add the two public hostnames `<DOMAIN>` and `*.<DOMAIN>` → Traefik, with a `http_status:404` catch-all (§4.2).
- DNS: proxied CNAMEs `@` and `*` → `<TUNNEL_ID>.cfargotunnel.com`; turn on Always Use HTTPS.
- Deploy Homepage at `<DOMAIN>` as the first app (§4.9).
- Confirm `https://<DOMAIN>` shows the Access login, then the dashboard.
- Confirm an unknown `https://<random>.<DOMAIN>` returns Traefik's `404` after login.
- Confirm no inbound `80`/`443` is open on the VM.
- Optional: install cert-manager only if internal TLS is needed.

**Done when:** Homepage is reachable at `https://<DOMAIN>` behind Access, and the VM's public IP serves nothing.

### Phase 4 — Secrets

- Create Infisical project/environment.
- Create a machine identity for ESO.
- Install ESO.
- Create `SecretStore` with Infisical Universal Auth.
- Create a test `ExternalSecret`; confirm a Kubernetes Secret appears.
- Document all bootstrap secrets outside Git.

**Done when:** a pod reads a secret sourced from Infisical, and no plaintext secret is in Git.

### Phase 5 — n8n and Nextcloud

- Deploy Postgres (or MariaDB) for each app as needed.
- Deploy n8n with:
  - `N8N_HOST`, `N8N_PROTOCOL=https`, `N8N_WEBHOOK_URL`
  - `N8N_PROXY_HOPS=1`
  - `N8N_ENCRYPTION_KEY` from Infisical
- Deploy Nextcloud with a local-path PVC; the provisioner stores it under `<DATA_VOLUME_MOUNT>/k3s/storage`.
- Add one standard `Ingress` per app with `gethomepage.dev/*` annotations, including `href` (§4.10). No DNS or tunnel change is needed.
- Verify data survives pod restart.
- Configure n8n webhook path/Access bypass.

**Done when:** both apps are reachable, persistent, shown as dashboard cards, and their secrets come from Infisical.

### Phase 6 — Access control

- The Cloudflare Access gate exists from Phase 3; tighten per-app policies and add the public-path bypasses.
- Add Authelia only if app-level SSO is required.
- Configure OIDC client(s) and ForwardAuth.
- Add Access bypass for the Authelia auth hostname/OIDC endpoints.
- Verify no double-login loop.
- Restrict Argo CD/Headlamp to admin only.

**Done when:** admin surfaces require auth and app login flow completes without loops.

### Phase 7 — Backups

- Assign OCI backup policy to boot + data volumes.
- Create Object Storage bucket and restic repository.
- Create `pg_dump` scripts/CronJobs and restic wrapper.
- Store restic password and encryption keys in a password manager.
- Run first backup; run first restore test.

**Done when:** a restore of one app succeeds on a scratch environment.

### Phase 8 — Optional components

Defer until core is stable:

- Zot registry
- self-hosted GitHub Actions runner
- AdGuard split-horizon DNS
- Headlamp
- Velero
- monitoring stack

**Done when:** core has run cleanly for a defined period and resource headroom is confirmed.

---

## 10. Resource budget (small single-node VM)

Approximate **requests**, not limits. Adjust to the actual node size:

| Component | CPU request | Memory request |
|---|---|---|
| OS + k3s | 0.25 | 1 GiB |
| Argo CD | 0.20 | 512 MiB |
| Traefik | 0.10 | 128 MiB |
| cloudflared | 0.05 | 64 MiB |
| Homepage | 0.05 | 128 MiB |
| ESO | 0.10 | 128 MiB |
| Authelia | 0.10 | 128 MiB |
| n8n | 0.25 | 512 MiB |
| Nextcloud | 0.25 | 512 MiB |
| Postgres | 0.25 | 512 MiB |
| Redis | 0.10 | 128 MiB |
| **Total** | **~1.70** | **~3.7 GiB** |

CPU is the tight resource, not RAM. Avoid building container images on the VM; use GitHub-hosted runners or defer the self-hosted runner. Infisical Cloud keeps the secrets stack off the VM; self-hosting adds ~0.3 CPU / 1 GiB plus Postgres/Redis.

---

## 11. Security checklist

- [ ] SSH key-only; no password login.
- [ ] SSH restricted to the admin source or Tailscale.
- [ ] Inbound `80`/`443`/`6443` closed.
- [ ] Cloudflare Tunnel only; no A record to the VM.
- [ ] Cloudflare Access gate on `<DOMAIN>` and `*.<DOMAIN>` before hostnames go live (Homepage has no built-in auth).
- [ ] `HOMEPAGE_ALLOWED_HOSTS` lists `<DOMAIN>` explicitly; never `*`.
- [ ] Authelia configured without double-login loops.
- [ ] No plaintext secrets in Git.
- [ ] Infisical machine identity least privilege.
- [ ] NetworkPolicies default-deny; allow DNS and required ingress/egress.
- [ ] Non-root containers; drop capabilities; read-only root FS where possible.
- [ ] Pin image tags/digests; no `latest`.
- [ ] Resource requests/limits set.
- [ ] Rate limiting at Cloudflare/Traefik.
- [ ] Audit logs retained.
- [ ] Backup credentials separate from app credentials.
- [ ] Restore tests scheduled.
- [ ] Regular updates and CVE checks; verify ARM64 images.
- [ ] `kubeconfig` mode `600`; API not publicly exposed.
- [ ] Data volume mounted by UUID and k3s guarded against a missing mount.

---

## 12. Risks and open questions

### Risks

- **Single node:** no HA; maintenance and node failure cause downtime.
- **CPU:** contention is likely; avoid heavy builds and monitoring stacks.
- **RAM:** enough for core, but self-hosted Infisical + monitoring can squeeze it.
- **local-path storage:** no replication; volume loss means data loss without backups.
- **Cloudflare dependency:** Tunnel/Access outage affects public access.
- **Infisical dependency:** if self-hosted and down, ESO cannot fetch new secrets; existing Kubernetes Secrets persist.
- **Bootstrap secret loss:** losing the Infisical key, restic password, or k3s token complicates recovery.
- **OCI idle reclamation:** Oracle may reclaim an Always Free instance it deems idle — over 7 days, 95th-percentile CPU < 20%, network < 20%, and (A1 shapes) memory < 20%. With only k3s system pods running, memory use is well below 20%, so the VM is idle-eligible until workloads run. Upgrading the account to Pay As You Go is the commonly cited way out; Oracle does not charge for Always Free resources after an upgrade.
- **Cloudflare Free limits:** 100 MB request cap and CDN terms on video and large files (§4.7).
- **ARM64 images:** validate every image before deployment.
- **Upgrades:** k3s, Argo CD, Traefik, ESO, and apps need a tested upgrade path.

### Open decisions

| Decision | Options | Recommendation | Status |
|---|---|---|---|
| Domain/registrar | Own/buy; Cloudflare DNS | Use one domain; move nameservers to Cloudflare | Pending |
| Public access | Tunnel vs Tailscale only | Tunnel + Tailscale admin | Adopted |
| App hostnames | Per-app tunnel hostnames vs wildcard | `<DOMAIN>` + `*.<DOMAIN>` → Traefik | Adopted |
| Load balancer | MetalLB, ServiceLB, OCI LB, none | None — tunnel to Traefik `ClusterIP` | Adopted |
| Dashboard | Homepage vs others | Homepage with `Ingress` autodiscovery | Adopted |
| Image updates | Renovate PRs vs Argo CD Image Updater | Renovate; automerge patch updates of stateless apps only | Pending |
| Infisical | Cloud vs self-host | Cloud free tier first | Pending |
| Git repo | New private vs existing | New private GitOps repo | Pending |
| Backups | OCI volume only vs + restic | OCI + restic + dumps | Pending |
| Auth | Access only vs Authelia | Access gate in Phase 3; Authelia later | Pending |
| Scope | Core vs full article | Core first; defer optional | Confirmed |
| Resources | Stay small vs resize | Stay on the existing resource class | Confirmed |
| Storage topology | Separate volumes vs merged pool | Keep separate; k3s data-dir on the data volume | Confirmed (implemented in Phase 1) |

---

## 13. Discussion log

- **Goal:** deploy n8n, Nextcloud, and other open-source apps on the OCI ARM VM using the article's k3s/Argo CD GitOps architecture.
- **Constraint:** no VM, repo, or memory changes without explicit approval; read-only by default.
- **Constraint:** follow the article's steps/stack; do not fall back to Docker Compose.
- **Inspected:** repository, SSH setup, VM OS/resources/storage/network.
- **Verified:** a single-node ARM VM with a separate ext4 block volume mounted at `<DATA_VOLUME_MOUNT>` via `/etc/fstab` (UUID, `_netdev,nofail`), ample free space, and no preinstalled Docker/k3s; the volume is empty except `lost+found`. Exact host identifiers are intentionally omitted.
- **Recommendation:** keep the boot and data volumes separate; do not merge them with LVM, RAID 0, or a union mount. Use the data volume for k3s state, images, and PVCs.
- **Researched:** stack options, ARM64 support, OCI access, reclaim rules, and current docs.
- **Article analysis:** k3s, Argo CD app-of-apps, Traefik, cert-manager, ESO + Infisical, Cloudflare Tunnel, Authelia, Zot, GitHub runner, NetworkPolicies.
- **Decision:** stay on the existing small single-node resource class.
- **Decision:** phased article stack; defer Zot/runner/AdGuard/Headlamp until core works.
- **Decision:** Docker Compose is superseded and rejected.
- **Diagram:** created a 58-element canvas covering domain setup, public/private access, GitOps/secrets, and backups; the share link is intentionally not embedded here.
- **Q&A — domain:** one domain is enough; do not point it at the public IP; move nameservers to Cloudflare; create a Tunnel; add CNAMEs to `<TUNNEL_ID>.cfargotunnel.com`; close inbound ports.
- **Q&A — Infisical:** secrets manager; ESO pulls secrets into Kubernetes Secrets; bootstrap secrets remain manual; Cloud free tier recommended for phase 1.
- **Q&A — Argo CD:** runs in cluster; bootstrap install is outside Argo CD; app-of-apps references Git; app manifests live in Git, not inside Argo CD; Argo CD can manage itself after bootstrap.
- **Q&A — backups:** OCI volume backups are crash-consistent, optional, and not application-consistent; GitOps does not back up PVC data; add `pg_dump` + restic + restore tests.
- **Phase 1 executed:** installed k3s `v1.36.5+k3s1` with the data volume as its data-dir, disabled bundled Traefik and ServiceLB, set kubeconfig mode `600`, capped container logs, and installed the systemd mount guard. The node is `Ready`; CoreDNS, local-path-provisioner, and metrics-server are healthy.
- **PVC smoke test passed:** the test PVC wrote and read a file under `<DATA_VOLUME_MOUNT>/k3s/storage`; the PVC, PV, and pod were then removed and the storage directory was empty.
- **`kubectl` correction:** bare `sudo kubectl` recreated a duplicate default runtime-assets directory on the root disk. Use the data-dir-aware command from §7.1. The duplicate was removed after confirming no open files or processes, and the cluster stayed healthy.
- **Q&A — dashboard (2026-10-21):** `<DOMAIN>` opens Homepage; cards come from `Ingress` annotations and link to `<app>.<DOMAIN>`. The tunnel publishes only `<DOMAIN>` and `*.<DOMAIN>`; Access gates both (§4.8–§4.10).
- **Q&A — load balancer:** no MetalLB, ServiceLB, or OCI Load Balancer; `cloudflared` connects outbound to Traefik's `ClusterIP` (§4.8).
- **Q&A — JobLens AI architecture:** reuse its `cloudflared` Deployment, tunnel API automation, and Argo CD `Application` pattern; drop KinD, MetalLB, the nginx `LoadBalancer` gateway, Gitea, and Vault.
- **Q&A — image updates:** Argo CD syncs Git, not registries. Renovate bumps pinned tags by PR; after merge, Argo CD deploys within about three minutes (§6.7).
- **Read-only VM check (2026-10-21):** node `Ready`; bundled Traefik and ServiceLB disabled; only CoreDNS, local-path-provisioner, and metrics-server running; nothing listening on `80`/`443`; the host firewall accepts new inbound connections only on SSH.
- **Verified in source/docs:** Homepage v2.6.0 builds `https` card links only for Ingresses with a `tls:` block (§4.10); Traefik chart v42.1.0 sets the Service type under `service.spec.type` (§7.3).
- **Pending:** domain and Cloudflare nameservers, Infisical hosting, Git repo location, backup bucket, and Authelia timing.

---

## 14. Article gaps and adaptations

The article is a good architecture reference, but several details are homelab-specific or incomplete:

1. **Backups:** no explicit application-data backup layer. Add OCI volume backups + app-consistent dumps.
2. **AdGuard split-horizon:** assumes a private LAN and local DNS. Not directly applicable to a cloud VM.
3. **LAN ForwardAuth:** assumes LAN clients; use Cloudflare Access/Tailscale instead.
4. **Traefik service type:** article describes ClusterIP but also mentions NodePort. Resolve as `ClusterIP` when routing through `cloudflared`.
5. **Cert-manager wildcard:** useful for internal TLS, but not required for public HTTPS through Cloudflare Tunnel.
6. **Cloudflare Tunnel config:** token mode keeps hostnames in the dashboard, not Git. Accept for phase 1; move to locally-managed config later if GitOps purity matters.
7. **Infisical bootstrap:** article glosses over the chicken-and-egg secret. Document and automate bootstrap secrets explicitly.
8. **Resource fit:** the article's full stack is tight on a small single-node VM; defer optional components.

---

## 15. References

- Article (Freedium mirror):  
  https://freedium-mirror.cfd/https://medium.com/@devidence/from-docker-compose-to-gitops-how-i-rebuilt-my-homelab-with-kubernetes-proper-secrets-and-a-full-065ed948801e
- k3s configuration options (data-dir and kubelet args):  
  https://docs.k3s.io/installation/configuration
- Cloudflare Tunnel DNS records:  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/dns/
- Cloudflare Tunnel with firewall (egress `7844`):  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/
- Infisical pricing:  
  https://infisical.com/pricing
- ESO Infisical provider:  
  https://external-secrets.io/latest/provider/infisical/
- OCI Block Volume backups:  
  https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/blockvolumebackups.htm
- OCI backup policies:  
  https://docs.oracle.com/en-us/iaas/Content/Block/Tasks/schedulingvolumebackups.htm
- n8n webhook URL behind reverse proxy:  
  https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/
- Homepage Kubernetes installation:  
  https://gethomepage.dev/installation/k8s/
- Homepage Kubernetes configuration and annotations:  
  https://gethomepage.dev/configs/kubernetes/
- Homepage card URL scheme logic (v2.6.0):  
  https://github.com/gethomepage/homepage/blob/v2.6.0/src/utils/kubernetes/resource-helpers.js
- Unofficial Homepage Helm chart index:  
  https://jameswynn.github.io/helm-charts/index.yaml
- MetalLB cloud compatibility:  
  https://metallb.io/installation/clouds/
- Oracle MetalLB lab (Layer 2 VLAN requirement):  
  https://docs.oracle.com/en/learn/ocne-metallb
- OCI Always Free resources (Load Balancer limits, idle reclamation):  
  https://docs.oracle.com/en-us/iaas/Content/FreeTier/resourceref.htm
- Cloudflare wildcard DNS records:  
  https://developers.cloudflare.com/dns/manage-dns-records/reference/wildcard-dns-records/
- Cloudflare Universal SSL limitations:  
  https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/
- Cloudflare Tunnel configuration file (wildcard ingress rules):  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/configuration-file/
- Cloudflare plans (Zero Trust Free: up to 50 users):  
  https://www.cloudflare.com/plans/
- Cloudflare request body limits by plan:  
  https://developers.cloudflare.com/workers/about/limits
- Cloudflare terms update (video and large files on the CDN):  
  https://blog.cloudflare.com/updated-tos/
- Traefik Helm chart values (v42.1.0):  
  https://github.com/traefik/traefik-helm-chart/blob/v42.1.0/traefik/values.yaml
- Argo CD Git polling and webhooks:  
  https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/
- Argo CD Image Updater:  
  https://argocd-image-updater.readthedocs.io/en/stable/
- Renovate Argo CD manager:  
  https://docs.renovatebot.com/modules/manager/argocd/

---

## 16. Next action

This plan lives at `docs/OCI-K3S-GITOPS-PLAN.md`.

Phase 1 is complete. Next:

1. Resolve the Phase 0 inputs:
   - domain/registrar and Cloudflare nameservers
   - Infisical Cloud vs self-hosted
   - private GitOps repo URL and credentials
   - OCI Object Storage bucket/region for restic
   - Authelia timing
2. Bootstrap Argo CD. This can be installed before the domain is ready; use `kubectl port-forward` for initial admin access.
3. Add the app-of-apps root once the private GitOps repo exists.
4. After the domain is active: create the Access gate, install Traefik, deploy `cloudflared` with the `<DOMAIN>` and `*.<DOMAIN>` hostnames, then deploy Homepage as the first app (Phase 3).
5. Install ESO/Infisical after the Infisical decision.
6. Add backup/restore checks before stateful apps.

Each VM write step requires explicit approval.
