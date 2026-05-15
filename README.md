# secrets-generator

An ArgoCD compatible Helm chart that generates Kubernetes Secrets with random credentials *at deploy time*. Secrets are created idempotently — existing keys are never overwritten, and only missing keys are added. Also works with plain `helm install`/`helm upgrade`

## Why Use This Chart

ArgoCD renders Helm charts with `helm template` — a purely client-side operation. This means Helm's built-in random functions (`randAlphaNum`, `genPrivateKey`) produce **new values on every sync**, making them unusable for stable secrets. The standard workaround is to use `lookup` to check for existing values, but `lookup` is not available during `helm template` and is therefore incompatible with ArgoCD.

This chart solves the problem by moving secret generation into a **Kubernetes Job** that runs as an ArgoCD PreSync hook. The Job executes `helm template` server-side (inside the cluster), generates the values once, and applies them idempotently — existing secrets are never modified. This gives you declarative, Git-driven secret definitions without committing actual secret values and without requiring external infrastructure.

### Use Cases

- **ArgoCD-managed applications** that need secrets generated at deploy time without manual intervention or external secret stores — the PreSync hook guarantees secrets exist before any application pod is reconciled
- **JWT/mTLS signing keys** that must be generated once per cluster and remain stable across ArgoCD syncs
- **Internal service passwords** (database credentials, inter-service tokens) that only need to be unique per environment, not centrally managed
- **Air-gapped and on-premise deployments** where cloud secret managers (AWS Secrets Manager, GCP Secret Manager) are unavailable
- **Ephemeral environments** (dev, staging, preview) that need unique secrets provisioned automatically as part of a GitOps workflow

## How It Works

A Kubernetes Job runs as a pre-install/pre-upgrade hook (compatible with both Helm and ArgoCD). The Job:

1. Renders secret values using Helm template functions (`randAlphaNum`, `genPrivateKey`, etc.)
2. For each defined secret:
   - If it **does not exist** — creates it
   - If it **exists but is missing keys** — patches in only the missing keys
   - If it **exists with all keys** — does nothing

This means secrets are stable across upgrades. Values generated on first install are preserved, and removing a secret from `values.yaml` does not delete it from the cluster.

## Installation

```bash
helm install secrets-generator . -n <namespace>
```

## Configuration

### `values.yaml` Structure

```yaml
image:
  repository: dtzar/helm-kubectl
  tag: "3.14"
  pullPolicy: IfNotPresent

serviceAccount:
  name: secrets-generator

secrets:
  - name: <kubernetes-secret-name>
    data:
      <key>: <value-or-helm-expression>
```

### Value Types

Values can be either **Helm template expressions** (evaluated at deploy time) or **literal strings** (stored as-is):

```yaml
secrets:
  # Random password generated on first install
  - name: my-app-credentials
    data:
      password: "{{ randAlphaNum 32 }}"
      api-key: "{{ randAlphaNum 64 }}"

  # RSA private keys
  - name: signing-keys
    data:
      private-key: |
        {{ genPrivateKey "rsa" }}
      ec-key: |
        {{ genPrivateKey "ec" }}

  # Literal values (stored directly, not generated)
  - name: static-config
    data:
      database-host: "postgres.internal"
      port: "5432"
```

### Available Helm Template Functions

**Any Helm template function can be used in secret values**, some examples:

| Expression                      | Description                             |
|---------------------------------|-----------------------------------------|
| `{{ randAlphaNum 32 }}`         | Random 32-character alphanumeric string |
| `{{ randAlpha 16 }}`            | Random 16-character alphabetic string   |
| `{{ genPrivateKey "rsa" }}`     | RSA private key (PEM)                   |
| `{{ genPrivateKey "ec" }}`      | EC private key (PEM, faster than RSA)   |
| `{{ genPrivateKey "ed25519" }}` | Ed25519 private key (PEM)               |
| `{{ randBytes 32 \| b64enc }}`  | 32 random bytes, base64-encoded         |

## Example

```yaml
# values.yaml
secrets:
  - name: app-secrets
    data:
      JWT_SECRET: "{{ randAlphaNum 64 }}"
      SESSION_KEY: "{{ randAlphaNum 32 }}"
      DB_PASSWORD: "{{ randAlphaNum 24 }}"

  - name: tls-signing
    data:
      SIGNING_KEY: |
        {{ genPrivateKey "ec" }}

  - name: external-api
    data:
      API_ENDPOINT: "https://api.example.com"
      API_VERSION: "v2"
```

```bash
# Install
helm install secrets-generator . -n my-namespace

# Verify
kubectl get secrets -n my-namespace

# Add a new key to app-secrets (next upgrade will add it without regenerating existing keys)
# Just add the key to values.yaml and run:
helm upgrade secrets-generator . -n my-namespace
```

## Comparison with Secrets Operators

This chart takes a fundamentally different approach from operators like [External Secrets Operator](https://external-secrets.io/) (ESO), [Sealed Secrets](https://sealed-secrets.netlify.app/), or [Vault Secrets Operator](https://developer.hashicorp.com/vault/docs/platform/k8s/vso).

### Advantages

|                            | secrets-generator                                                              | Secrets Operators (ESO, etc.)                                                   |
|----------------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| **Zero infrastructure**    | Just a Helm chart. No CRDs, no controllers, no running pods                    | Requires a cluster-wide operator with CRDs, RBAC, and continuous reconciliation |
| **Simple setup**           | Add secrets to `values.yaml` and deploy                                        | Configure operator + SecretStore + ExternalSecret resources per secret          |
| **Offline/air-gapped**     | Works without network access to external services                              | Depends on connectivity to the secret provider                                  |
| **Cost**                   | Free — no cloud secrets manager billing                                        | External stores often have per-secret or per-API-call costs                     |

### Disadvantages

|                                     | secrets-generator                                                             | Secrets Operators (ESO, etc.)                                                    |
|-------------------------------------|-------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| **No rotation**                     | Values are generated once and never rotated automatically                     | Can rotate secrets on a schedule or when the source changes                      |
| **No central management**           | Secrets are scoped to the cluster — no single pane of glass                   | Central store provides audit logs, access policies, and cross-cluster visibility |
| **Not suitable for shared secrets** | Each install generates unique values — cannot share a secret across clusters  | A central store serves the same secret to multiple clusters                      |
| **Random only**                     | Generates random values or static strings — cannot fetch existing credentials | Can sync secrets from any supported backend (databases, APIs, certificates)      |

### When to Use This Chart

- As sub-chart for Helm Charts not supporting propert generation of secrets 
- Installations that need unique secrets without external infrastructure
- Self-hosted / air-gapped deployments where cloud secret managers are unavailable
- Bootstrap secrets (signing keys, internal passwords) that are generated once and don't need rotation
- Situations where adding an operator is disproportionate to the number of secrets managed

### When to Use a Secrets Operator Instead

- You're already using a Secrets Operator in your cluster
- Environments requiring secret rotation
- Secrets that originate from an external system (API keys, database credentials provisioned elsewhere)
- Multi-cluster deployments sharing the same secret values
- Compliance requirements mandating centralized secrets management

## Security

- Container runs as non-root (UID 1000) with read-only root filesystem
- Temporary files stored in memory-backed emptyDir (never written to disk)
- All rendered secrets cleaned up on Job exit
- Input validation on secret names, key names, and values
- RBAC scoped to secrets only (get, create, patch, delete)
- PodSecurity `restricted:latest` compliant
