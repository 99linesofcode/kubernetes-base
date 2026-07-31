# kubernetes-base

Umbrella Helm chart for deploying a FrankenPHP/Laravel application stack on Kubernetes — k3s for local development, any production cluster via Flux GitOps. The chart composes all infrastructure dependencies (Traefik, cert-manager, PostgreSQL, Redis) and the FrankenPHP application subchart into a single installable unit.

Repository: <https://github.com/99linesofcode/kubernetes-base>

---

## Table of Contents

1. [Conceptual Overview](#conceptual-overview)
   - [What This Stack Is](#what-this-stack-is)
   - [Architecture at a Glance](#architecture-at-a-glance)
   - [Component Reference](#component-reference)
   - [Secrets Flow](#secrets-flow)
   - [Configuration Flow](#configuration-flow)
   - [Deployment Flow](#deployment-flow)
2. [Practical Usage Guide](#practical-usage-guide)
   - [Prerequisites](#prerequisites)
   - [Local Setup (k3s)](#local-setup-k3s)
   - [Running the Laravel App Against the Cluster](#running-the-laravel-app-against-the-cluster)
   - [Testing and Verification](#testing-and-verification)
   - [Adding a New Component to the Umbrella Chart](#adding-a-new-component-to-the-umbrella-chart)
   - [Changing Configuration: Local vs Production](#changing-configuration-local-vs-production)
   - [Managing Secrets with SOPS](#managing-secrets-with-sops)
   - [Common Operations](#common-operations)
   - [Troubleshooting](#troubleshooting)

---

## Conceptual Overview

> High-level explanation — no code. For concrete commands and YAML, see the [Practical Usage Guide](#practical-usage-guide).

### What This Stack Is

This stack deploys a complete Laravel application running on FrankenPHP (a modern PHP app server built on Caddy) to a Kubernetes cluster. It handles the full set of concerns a production Laravel app needs: HTTP entry and TLS termination, certificate issuance, a relational database, a cache/queue store, and GitOps-driven declarative deployments.

The same chart serves both environments — a single-node k3s cluster on a developer's laptop for local development, and a multi-node production cluster managed by Flux CD. Environment differences are expressed purely through values files and HelmRelease overlays, never through template branching. This keeps the chart portable and the templates clean.

The key problem this solves: running a real Laravel application on Kubernetes with production-grade infrastructure (Gateway API routing, automated TLS, encrypted secrets in git) while keeping the local development loop fast — live code reloading via hostPath volume mounts, no image rebuilds needed during development.

### Architecture at a Glance

The stack is an umbrella chart named `laravel-stack`. It composes five subcharts, each owning one concern:

- **Traefik** (vendored chart, v36.2.0) — the Gateway API controller. It owns the Gateway and GatewayClass resources, terminates TLS, and routes HTTP traffic. It runs in hostNetwork mode on k3s to bind directly to ports 80 and 443 on the host. No Kubernetes Service is created for Traefik itself — hostNetwork makes one unnecessary.

- **cert-manager** (Jetstack chart, v1.17.2) — automates TLS certificate lifecycle. It creates and renews the TLS certificate that Traefik uses for the HTTPS listener. In local dev it uses a self-signed CA chain (a self-signed issuer bootstraps a local CA, then a CA issuer signs the leaf cert). In production it uses Let's Encrypt with a Cloudflare DNS-01 challenge.

- **PostgreSQL** (Bitnami chart, v16.7.2) — the database. Conditionally enabled via `postgresql.enabled`. In local dev it runs without persistence (ephemeral storage, no password for the user). In production it runs with persistent volumes and passwords sourced from SOPS-encrypted secrets.

- **Redis** (Bitnami chart, v27.0.18) — cache and queue backend. Conditionally enabled via `redis.enabled`. Local dev: no auth, no persistence. Production: auth enabled, persistence on, password from SOPS.

- **FrankenPHP** (local subchart at `charts/frankenphp/`) — the application itself. A Deployment running the FrankenPHP Docker image, a ClusterIP Service, and an HTTPRoute that attaches to the Gateway's websecure listener. In local dev it mounts the host's code directory via hostPath for live reloading. In production the code is baked into the image — no hostPath.

The umbrella chart also owns three infrastructure-level templates that are not part of any subchart:

- **ClusterIssuers** — three separate template files, each gated by a values toggle: `self-signed-issuer.yaml` (bootstraps a self-signed issuer), `local-ca-issuer.yaml` (creates a CA Certificate from that issuer, then a CA ClusterIssuer that uses it), and `cloudflare-issuer.yaml` (creates a Let's Encrypt ACME ClusterIssuer with Cloudflare DNS-01). Only the enabled issuers are rendered — the values files toggle them per environment.

- **Certificate** — a single `certificate.yaml` template requests a TLS cert for `global.domain` from the issuer named in `global.issuer.name`. The resulting Secret (`frankenphp-tls`) is referenced by the Gateway's websecure listener.

- **HTTP-to-HTTPS redirect** — `httproute-redirect.yaml` creates an HTTPRoute attached to the Gateway's `web` (HTTP) listener. It uses the Gateway API standard `RequestRedirect` filter to issue a 301 redirect to HTTPS on port 443. This is portable across any Gateway API controller, not Traefik-specific.

Traffic flow: a client request hits port 80 or 443 on the host (Traefik in hostNetwork mode). HTTP on port 80 is redirected to HTTPS on port 443. HTTPS is terminated at Traefik using the cert from cert-manager. Traefik then routes to the FrankenPHP Service via the app's HTTPRoute (attached to the `websecure` listener). FrankenPHP serves the Laravel application from `/app/public`. In local dev, `/app` is the hostPath mount of the developer's code directory. In production, `/app` is baked into the Docker image.

### Component Reference

**Traefik** — why it's here: we need a Gateway API controller that can terminate TLS and route HTTP traffic. Traefik v3 implements the Kubernetes Gateway API spec and is well-supported on k3s (it's actually the default ingress controller shipped with k3s, though we use it in Gateway API mode, not Ingress mode). Role in the pipeline: edge of the cluster — receives all external HTTP/S traffic, handles TLS, routes to backends.

**cert-manager** — why it's here: TLS certificates need to be issued and renewed automatically. cert-manager integrates with the Gateway API (it can provision certs referenced by Gateway listeners) and supports both self-signed (local) and ACME/DNS-01 (production) issuance. Role: sits behind Traefik, creates the `frankenphp-tls` Secret that the Gateway references. In production it talks to Let's Encrypt and Cloudflare's API to complete DNS-01 challenges.

**PostgreSQL** — why it's here: Laravel needs a relational database. The Bitnami chart gives us a production-ready PostgreSQL with configurable persistence, auth, and replication. Role: data store for the application. The FrankenPHP deployment gets a `DATABASE_URL` environment variable pointing to the PostgreSQL Service.

**Redis** — why it's here: Laravel uses Redis for cache, sessions, and queue. The Bitnami chart provides standalone or replicated Redis with optional auth and persistence. Role: cache and queue backend. The FrankenPHP deployment gets a `REDIS_URL` environment variable pointing to the Redis master Service.

**FrankenPHP subchart** — why it's here: this is the application. FrankenPHP is a PHP application server built on Caddy that runs PHP workers in a single process with very low overhead. Role: serves HTTP responses for the Laravel application. The subchart is vendored locally at `charts/frankenphp/` (a `file://` dependency), so it's always available and doesn't need a separate registry pull during `helm dep build`.

**Flux CD** (lives in the separate `kubernetes-fleet` repo, not in this chart) — why it's here: production deployments are GitOps-driven. Flux watches the fleet repo for changes and reconciles them into the cluster. Role: pulls the umbrella chart from an OCI registry, applies it with production values, and decrypts SOPS-encrypted secrets before the chart references them.

**SOPS** (age encryption) — why it's here: secrets must be encrypted at rest in git. SOPS encrypts the secret values in the YAML files; Flux's kustomize-controller decrypts them in-cluster during reconciliation using an age private key stored as a Kubernetes Secret. Role: the bridge between "secrets in git" and "secrets in the cluster" without ever storing plaintext in the repository.

### Secrets Flow

Secrets never appear as plaintext in git. The full chain:

1. A developer writes a Kubernetes Secret manifest with real values (database passwords, Cloudflare API token).
2. The developer runs `sops --encrypt --in-place` on the file. SOPS uses the age public key from `.sops.yaml` to encrypt the secret values. The file is committed to git in its encrypted form — only the metadata (name, namespace, keys) remains readable, the values are ciphertext.
3. Flux's GitRepository source-controller pulls the fleet repo (which contains the encrypted secret files under `secrets/production/`).
4. Flux's kustomize-controller reconciles the `sops-secrets` Kustomization, which has `decryption.provider: sops`. It reads the age private key from the `sops-age-key` Secret in the `flux-system` namespace and decrypts the files in-memory.
5. The decrypted Kubernetes Secret is applied to the cluster in the `default` namespace (or wherever `targetNamespace` points).
6. The umbrella chart's HelmRelease (`laravel-skeleton-production`) has `dependsOn: sops-secrets`, so Flux waits until the Secrets exist before applying the chart.
7. The chart's values reference the pre-created Secret via `auth.existingSecret: app-secrets` for both PostgreSQL and Redis. The Bitnami charts read passwords from that Secret instead of generating their own.
8. The FrankenPHP deployment template references the same `app-secrets` Secret for `POSTGRES_PASSWORD` and `REDIS_PASSWORD` environment variables (with `optional: true` so local dev without auth doesn't break).

Two age key pairs are used — one for local, one for production. The `.sops.yaml` file at each repo root maps path regexes to age recipients, so `sops` automatically picks the right key. Production secrets cannot be decrypted with the local key, and vice versa.

### Configuration Flow

Configuration is layered. Three values files stack on top of each other, and Flux HelmReleases add another layer:

1. **`values.yaml`** — the base layer. Defines defaults shared across all environments: the Gateway name and namespace, the domain (defaulting to the local `.test` domain), Traefik listener configuration, cert-manager CRD installation, PostgreSQL/Redis defaults (disabled by default, existingSecret set to `app-secrets`), and FrankenPHP defaults (image, service port, app secret name). This file is always loaded.

2. **`values.local.yaml`** — local dev overlay. Applied with `-f values.local.yaml` during local `helm install`/`helm upgrade`. Overrides: enables PostgreSQL (no persistence), enables Redis (no auth, no persistence), enables the self-signed and local CA issuers, disables the Cloudflare issuer, sets the hostPath for live code reloading, enables database and Redis env vars in FrankenPHP.

3. **`values.production.yaml`** — production overlay. Applied when installing for production (or via the production HelmRelease). Overrides: enables PostgreSQL with 10Gi persistence, enables Redis with auth and 5Gi persistence, disables self-signed/local CA issuers, enables the Cloudflare DNS-01 issuer, sets the production domain (`.nl`), uses the production Docker image, no hostPath.

4. **Flux HelmRelease overlays** — the fleet repo (`kubernetes-fleet`) contains two HelmReleases: `laravel-skeleton` (local) and `laravel-skeleton-production` (production). Each HelmRelease embeds its own `values:` block that overrides the chart's values. The production HelmRelease also has `dependsOn: sops-secrets` to ensure secrets are decrypted first. Flux applies these values on top of the chart's built-in `values.yaml` — the fleet's values are the authoritative overrides for each environment.

The key principle: templates are environment-agnostic. No `{{- if eq .Values.global.environment "local" }}` branching exists in the chart templates. Environment differences are expressed entirely through values toggles (`issuers.selfSigned.enabled`, `postgresql.enabled`, `frankenphp.hostPath`, etc.). The only place `global.environment` is used is in the values files themselves as documentation — the templates don't read it.

### Deployment Flow

**Local (direct Helm):**
1. Developer installs k3s on their machine.
2. `helm dependency build` fetches and vendors the external charts (Traefik, cert-manager, PostgreSQL, Redis) into `charts/`.
3. `helm install laravel-stack . -f values.local.yaml` installs the chart with local overrides.
4. Helm renders all templates, creates the resources in the cluster, and the subcharts create their respective resources.
5. Pods start: Traefik (hostNetwork, ports 80/443), cert-manager controller, PostgreSQL, Redis, FrankenPHP.
6. cert-manager issues the self-signed CA, then the leaf cert, creating the `frankenphp-tls` Secret.
7. The Gateway becomes `Programmed=True`, the HTTPRoute routes traffic, FrankenPHP serves the app.

**Production (Flux GitOps):**
1. The umbrella chart is packaged (`helm package`) and pushed to the OCI registry (`oci://ghcr.io/99linesofcode/charts`).
2. Flux is bootstrapped on the production cluster pointing at the `kubernetes-fleet` repo.
3. Flux's source-controller pulls the fleet repo. The kustomize-controller applies the root `kustomization.yaml`, which includes the `sops-secrets` Kustomization and the two HelmReleases.
4. The `sops-secrets` Kustomization decrypts the SOPS-encrypted secrets in `secrets/production/` using the age key and applies them to the cluster.
5. The production HelmRelease (`laravel-skeleton-production`) pulls the `laravel-stack` chart from the `ghcr-charts` HelmRepository (OCI registry). It waits for `sops-secrets` to complete first (`dependsOn`).
6. Flux's helm-controller installs the chart with the production values embedded in the HelmRelease.
7. Pods start. cert-manager requests a cert from Let's Encrypt via Cloudflare DNS-01. Traefik terminates TLS. FrankenPHP serves the app from the baked image.
8. On any future `git push` to the fleet repo, Flux detects the change, reconciles, and updates the cluster — typically within the 5-minute interval, or immediately with `flux reconcile`.

---

## Practical Usage Guide

### Prerequisites

Tools you need installed locally:

| Tool | Purpose | Install |
|------|---------|---------|
| `helm` | Render, lint, install the chart | <https://helm.sh/docs/intro/install/> |
| `kubectl` | Talk to the cluster | <https://kubernetes.io/docs/tasks/tools/> |
| `flux` | Bootstrap and reconcile Flux CD | `curl -s https://fluxcd.io/install.sh \| sudo bash` |
| `k3s` | Local single-node Kubernetes | `curl -sfL https://get.k3s.io \| sh -` |
| `sops` | Encrypt/decrypt secrets | <https://github.com/getsops/sops/releases> |
| `age` | Encryption backend for SOPS | <https://github.com/FiloSottile/age/releases> |

Set the kubeconfig for k3s:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### Local Setup (k3s)

**1. Install k3s:**
```bash
curl -sfL https://get.k3s.io | sh -
# Wait for it to be ready
sudo k3s kubectl get nodes
```

k3s ships with its own Traefik and ServiceLB. The umbrella chart installs Traefik via its own dependency (v36.2.0 in Gateway API mode), so disable k3s's bundled Traefik to avoid conflicts:
```bash
# Install k3s without the bundled Traefik
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

**2. Install Gateway API CRDs:**

The Gateway API CRDs are not bundled in the chart. Install them before the chart:
```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

**3. Clone and build dependencies:**
```bash
git clone https://github.com/99linesofcode/kubernetes-base.git
cd kubernetes-base
git checkout simple-gateway-api

# Fetch and vendor external chart dependencies (Traefik, cert-manager, PostgreSQL, Redis)
helm dependency build
```

This populates `charts/traefik-36.2.0.tgz`, `charts/cert-manager-v1.17.2.tgz`, `charts/postgresql-16.7.2.tgz`, and `charts/redis-27.0.18.tgz`.

**4. Generate an age key pair for local secrets:**
```bash
age-keygen -o age-local-key.txt
# Note the public key (recipient) printed to stderr
```

Update `.sops.yaml` with your local public key:
```yaml
creation_rules:
  - path_regex: secrets/local/.*\.ya?ml$
    age: <your-local-public-key>
```

**5. Create the local SOPS secret:**

Write a plaintext Secret, encrypt it, and verify:
```bash
# Create the secret file
cat > secrets/local/secrets.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: default
type: Opaque
stringData:
  password: your-local-db-password
  postgres-password: your-local-admin-password
EOF

# Encrypt with your local age key
export SOPS_AGE_KEY_FILE=./age-local-key.txt
sops --encrypt --in-place secrets/local/secrets.yaml

# Verify decryption
sops --decrypt secrets/local/secrets.yaml
```

**6. Apply the secret to the cluster (local — no Flux yet):**
```bash
# Decrypt and apply directly for local dev
sops --decrypt secrets/local/secrets.yaml | kubectl apply -f -
```

**7. Install the chart with local values:**
```bash
helm install laravel-stack . -f values.local.yaml
```

**8. Wait for all pods:**
```bash
kubectl get pods -w
# Expected: traefik, cert-manager, postgresql, redis, frankenphp all Running
```

### Running the Laravel App Against the Cluster

The local values mount your code directory into the FrankenPHP pod via hostPath. Edit `values.local.yaml` to point `frankenphp.hostPath` at your Laravel project:

```yaml
frankenphp:
  hostPath: /home/youruser/Development/laravel-skeleton
```

FrankenPHP serves PHP from `/app/public` inside the container, which maps to the `public/` directory of your Laravel project on the host. File changes are immediately visible — no image rebuild needed.

Add the local domain to your hosts file:
```bash
echo "127.0.0.1 frankenphp.99linesofcode.test" | sudo tee -a /etc/hosts
```

Access the app:
```bash
# HTTP redirects to HTTPS
curl -I http://frankenphp.99linesofcode.test
# Expected: 301 Moved Permanently, Location: https://...

# HTTPS serves the app (-k skips self-signed cert verification)
curl -k https://frankenphp.99linesofcode.test
# Expected: Laravel application response
```

To trust the local CA in your browser/OS (so HTTPS works without -k):
```bash
kubectl get secret frankenphp-ca-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > frankenphp-ca.crt

# Debian/Ubuntu:
sudo cp frankenphp-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Fedora/RHEL:
sudo cp frankenphp-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# macOS:
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain frankenphp-ca.crt
```

### Testing and Verification

**Lint the chart:**
```bash
helm lint .
# Expected: 0 chart(s) failed
```

**Render templates without installing (see what would be created):**
```bash
# Default values (minimal — PostgreSQL and Redis disabled)
helm template . > /tmp/render-default.yaml

# Local values
helm template . -f values.local.yaml > /tmp/render-local.yaml

# Production values
helm template . -f values.production.yaml > /tmp/render-production.yaml
```

**Verify the Gateway is ready:**
```bash
kubectl get gateway traefik-gateway
# Expected: ACCEPTED=True, PROGRAMMED=True
```

**Verify the TLS certificate:**
```bash
kubectl get certificate frankenphp-cert
# Expected: READY=True
```

**Verify HTTP redirects to HTTPS:**
```bash
curl -I http://frankenphp.99linesofcode.test
# Expected: HTTP/1.1 301 Moved Permanently
#           Location: https://frankenphp.99linesofcode.test/
```

**Check the database connection from the app pod:**
```bash
kubectl exec deploy/laravel-stack-frankenphp-deployment -- php artisan db:show
# Expected: database connection details, no errors
```

**Check Redis from the app pod:**
```bash
kubectl exec deploy/laravel-stack-frankenphp-deployment -- \
  php artisan tinker --execute="Redis::ping();"
# Expected: "PONG"
```

### Adding a New Component to the Umbrella Chart

This example adds a hypothetical **Meilisearch** search service to the stack.

**Step 1: Add the dependency to `Chart.yaml`:**
```yaml
dependencies:
  # ... existing dependencies ...
  - name: meilisearch
    version: 0.7.0
    repository: https://charts.bitnami.com/bitnami
    condition: meilisearch.enabled
```

The `condition` field makes it optional — it only renders when `meilisearch.enabled: true` in the values.

**Step 2: Add defaults to `values.yaml`:**
```yaml
# --- Meilisearch ---
meilisearch:
  enabled: false
  auth:
    existingSecret: app-secrets
    existingSecretPasswordKey: meilisearch-master-key
```

**Step 3: Enable it in `values.local.yaml`:**
```yaml
meilisearch:
  enabled: true
  persistence:
    enabled: false
```

**Step 4: Enable it in `values.production.yaml`:**
```yaml
meilisearch:
  enabled: true
  persistence:
    enabled: true
    size: 5Gi
```

**Step 5: Wire the env var into the FrankenPHP deployment** (`charts/frankenphp/templates/deployment.yaml`):
```yaml
{{- if .Values.meilisearch.enabled }}
- name: MEILISEARCH_PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ .Values.appSecretName }}
      key: meilisearch-master-key
      optional: true
- name: MEILISEARCH_URL
  value: "http://{{ .Release.Name }}-meilisearch:7700"
{{- end }}
```

Note: the frankenphp subchart needs a `meilisearch.enabled` toggle in its own `values.yaml` too, since subchart values are namespaced. Add `meilisearch.enabled: false` to `charts/frankenphp/values.yaml` and set it to `true` in the environment overrides.

**Step 6: Add the password to the SOPS secret:**
```bash
export SOPS_AGE_KEY_FILE=./age-local-key.txt
sops --set '["stringData"]["meilisearch-master-key"]="your-meili-key"' \
  secrets/local/secrets.yaml
```

**Step 7: Fetch the new dependency and test:**
```bash
helm dependency build
helm lint .
helm template . -f values.local.yaml | grep meilisearch
# Expected: StatefulSet, Service, ServiceAccount rendered
```

**Step 8: Upgrade the release:**
```bash
helm upgrade laravel-stack . -f values.local.yaml
kubectl get pods -l app.kubernetes.io/name=meilisearch
```

### Changing Configuration: Local vs Production

Environment differences are expressed through values files, not template branching. The toggles that differ:

| Setting | Local (`values.local.yaml`) | Production (`values.production.yaml`) |
|---------|---------------------------|--------------------------------------|
| `global.domain` | `frankenphp.99linesofcode.test` | `frankenphp.99linesofcode.nl` |
| `global.issuer.name` | `local-ca-issuer` | `cloudflare-dns01-issuer` |
| `issuers.selfSigned.enabled` | `true` | `false` |
| `issuers.localCA.enabled` | `true` | `false` |
| `issuers.cloudflare.enabled` | `false` | `true` |
| `postgresql.enabled` | `true` | `true` |
| `postgresql.primary.persistence` | `false` | `true` (10Gi) |
| `redis.enabled` | `true` | `true` |
| `redis.auth.enabled` | `false` | `true` |
| `redis.master.persistence` | `false` | `true` (5Gi) |
| `frankenphp.hostPath` | `/home/.../laravel-skeleton` | (omitted — baked image) |
| `frankenphp.image` | `dunglas/frankenphp:php8.5-alpine` | `ghcr.io/99linesofcode/frankenphp:v1.0.0` |
| `cloudflare.email` | (omitted) | `99linesofcode@gmail.com` |

To change a setting for a single install without editing files:
```bash
helm upgrade laravel-stack . -f values.local.yaml \
  --set frankenphp.image.tag=development
```

For Flux-managed production, edit the values block in the HelmRelease (`kubernetes-fleet/flux/apps/laravel-skeleton-production.yaml`) and push to git. Flux reconciles automatically.

### Managing Secrets with SOPS

The chart uses the `existingSecret` pattern: the Bitnami PostgreSQL and Redis charts read passwords from a pre-created Secret named `app-secrets` instead of generating their own. This Secret is created from SOPS-encrypted files.

**Encrypt a new secret file:**
```bash
export SOPS_AGE_KEY_FILE=./age-production-key.txt

# Write a plaintext Secret
cat > secrets/production/secrets.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: default
type: Opaque
stringData:
  password: your-db-user-password
  postgres-password: your-db-admin-password
  redis-password: your-redis-password
EOF

# Encrypt in place
sops --encrypt --in-place secrets/production/secrets.yaml
```

**Decrypt to verify:**
```bash
export SOPS_AGE_KEY_FILE=./age-production-key.txt
sops --decrypt secrets/production/secrets.yaml
```

**Edit an encrypted file (SOPS handles decrypt/edit/re-encrypt):**
```bash
export SOPS_AGE_KEY_FILE=./age-production-key.txt
sops secrets/production/secrets.yaml
# Opens $EDITOR with decrypted content; re-encrypts on save
```

**Add a new key without decrypting the whole file:**
```bash
sops --set '["stringData"]["new-key"]="new-value"' secrets/production/secrets.yaml
```

**For local dev (no Flux), apply the decrypted secret directly:**
```bash
export SOPS_AGE_KEY_FILE=./age-local-key.txt
sops --decrypt secrets/local/secrets.yaml | kubectl apply -f -
```

**For production (Flux), just commit and push** — Flux decrypts in-cluster:
```bash
git add secrets/production/secrets.yaml
git commit -m "chore: update production secrets"
git push
# Flux reconciles within 5m, or force it:
flux reconcile kustomization sops-secrets --with-source
```

See the SOPS setup guide in the deliverables folder for the full reference including key rotation.

### Common Operations

**Scale FrankenPHP pods:**
```bash
kubectl scale deploy/laravel-stack-frankenphp-deployment --replicas=3
```

**Check logs:**
```bash
# FrankenPHP
kubectl logs deploy/laravel-stack-frankenphp-deployment -f

# Traefik
kubectl logs -l app.kubernetes.io/name=traefik -f

# cert-manager
kubectl logs -l app.kubernetes.io/name=cert-manager -n cert-manager -f
```

**Port-forward a service (e.g., PostgreSQL for a DB GUI):**
```bash
kubectl port-forward svc/laravel-stack-postgresql 5432:5432
# Now connect to localhost:5432
```

**Restart a service (rollout restart):**
```bash
kubectl rollout restart deploy/laravel-stack-frankenphp-deployment
kubectl rollout restart statefulset/laravel-stack-postgresql
kubectl rollout restart statefulset/laravel-stack-redis-master
```

**Check the full set of resources created by the chart:**
```bash
kubectl get all -l app.kubernetes.io/instance=laravel-stack
```

**Force Flux reconciliation (production):**
```bash
flux reconcile kustomization flux-system --with-source
flux reconcile helmrelease laravel-skeleton-production -n flux-system
```

**Upgrade the chart after code changes:**
```bash
# Local
helm upgrade laravel-stack . -f values.local.yaml

# Production (via Flux — just push the new chart version to OCI)
helm package .
helm push laravel-stack-0.1.0.tgz oci://ghcr.io/99linesofcode/charts
# Then update the chart version in the fleet HelmRelease and push
```

### Troubleshooting

**Pods stuck in Pending:**
```bash
kubectl describe pod <pod-name>
# Check Events section for scheduling failures
# Common: no StorageClass for PVCs (local k3s has local-path by default)
# Common: hostPath directory doesn't exist on the node
```

**Gateway not Programmed:**
```bash
kubectl describe gateway traefik-gateway
# Check Conditions: ACCEPTED should be True, PROGRAMMED should be True
# If PROGRAMMED=False, check Traefik logs:
kubectl logs -l app.kubernetes.io/name=traefik -f
# Common: Gateway API CRDs not installed before Traefik started
# Fix: kubectl apply -f <gateway-api-crds> then restart Traefik
kubectl rollout restart deploy/traefik -n default
```

**Certificate not Ready:**
```bash
kubectl describe certificate frankenphp-cert
# Check Events for issuance failures
# Common (local): self-signed issuer not created — check:
kubectl get clusterissuer
# Common (production): Cloudflare API token invalid or DNS not pointing to cluster
kubectl get secret cloudflare-api-token -n cert-manager
sops --decrypt secrets/production/cloudflare-api-token.yaml
```

**FrankenPHP can't connect to PostgreSQL:**
```bash
# Check the DATABASE_URL env var in the pod
kubectl exec deploy/laravel-stack-frankenphp-deployment -- env | grep DATABASE
# Check the Secret exists
kubectl get secret app-secrets
# Check PostgreSQL is running
kubectl get pods -l app.kubernetes.io/name=postgresql
# Test connection
kubectl exec deploy/laravel-stack-frankenphp-deployment -- \
  php artisan db:show
```

**FrankenPHP can't connect to Redis:**
```bash
# Check the REDIS_URL env var
kubectl exec deploy/laravel-stack-frankenphp-deployment -- env | grep REDIS
# Check Redis is running
kubectl get pods -l app.kubernetes.io/name=redis
# Test connection (local dev has no auth, so REDIS_PASSWORD may be empty)
kubectl exec deploy/laravel-stack-frankenphp-deployment -- \
  php artisan tinker --execute="Redis::ping();"
```

**HTTP not redirecting to HTTPS:**
```bash
# Check the redirect HTTPRoute exists
kubectl get httproute http-to-https-redirect
# It should be attached to the web listener
kubectl describe httproute http-to-https-redirect
# Check the Gateway has a web listener
kubectl get gateway traefik-gateway -o jsonpath='{.spec.listeners[*].name}'
```

**SOPS decryption failing in Flux:**
```bash
# Check the age key Secret exists
kubectl get secret sops-age-key -n flux-system
# Check the Kustomization status
flux get kustomization sops-secrets
# Check kustomize-controller logs
kubectl logs -l app=source-controller -n flux-system -f
# Common: wrong key in the Secret, or .sops.yaml has wrong recipient
```

**helm dependency build fails:**
```bash
# Clear the charts directory and Chart.lock, rebuild
rm -rf charts/*.tgz Chart.lock
helm dependency build
# If behind a proxy:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency update
```

**Chart renders but resources are missing (PostgreSQL/Redis not created):**
```bash
# Check the condition toggle in your values
helm template . -f values.local.yaml | grep -A2 "postgresql"
# Ensure postgresql.enabled: true is in values.local.yaml
# Ensure the condition in Chart.yaml matches the values key exactly
```

---

## Related Documentation

- Architecture — full Helm chart architecture, ADRs, fitness functions (in the Obsidian deliverables folder: `kubernetes-frankenphp-stack/architecture.md`)
- Flux Setup Guide — bootstrapping Flux, OCI registry secrets, reconciliation (in the Obsidian deliverables folder: `kubernetes-frankenphp-stack/flux-setup.md`)
- SOPS Setup Guide — age key generation, encryption, Flux decryption, key rotation (in the Obsidian deliverables folder: `kubernetes-frankenphp-stack/sops-setup.md`)

## Contributing

Please review our [Contribution Guidelines](https://github.com/99linesofcode/.github/blob/main/.github/CONTRIBUTING.md).

## Code of Conduct

Please review and abide by the [Code of Conduct](https://github.com/99linesofcode/.github?tab=coc-ov-file).

## Security Vulnerabilities

Please review [the security policy](https://github.com/99linesofcode/.github?tab=security-ov-file) on how to report security vulnerabilities.

## License

This software is open source and licensed under the [MIT license](https://github.com/99linesofcode/kubernetes-base?tab=MIT-1-ov-file).
