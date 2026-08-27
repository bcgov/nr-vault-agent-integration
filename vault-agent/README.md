# Vault Agent chart

This chart is deployed once, separately from any application backend/frontend. It authenticates to Vault with AppRole and renders one or more KV secrets to files on a shared RWX PVC. It does not expose secrets over a network endpoint.

## Installing the published chart

Bumping `version` in [`Chart.yaml`](./Chart.yaml) on `main` triggers a tag, which publishes the packaged chart to GitHub Pages. Other repositories/applications can then install it without checking out this repo:

```sh
helm repo add vault-agent https://bcgov.github.io/nr-vault-agent-integration
helm repo update
helm upgrade --install vault-agent vault-agent/vault-agent -f my-values.yaml
```

Each application's OpenShift deployment mounts the resulting PVC (`<release-name>-secrets`) read-only; see "Consuming the shared secrets from another application" below.

Each entry in `vaultAgent.secrets` is rendered to its own file, so multiple applications can each get their own key/value set from a single vault-agent release without stepping on each other:

```yaml
vaultAgent:
  secrets:
    - name: appa
      secretPath: kv/data/appa/secret
      keys: [username, password]
      format: env            # KEY=value per line
      outputFile: /vault/output/appa.env
    - name: appb
      secretPath: kv/data/appb/secret
      keys: [token]
      format: json            # full secret data as JSON
      outputFile: /vault/output/appb.json
    - name: appc
      secretPath: kv/data/appc/secret
      keys: [value]
      format: raw              # single key's raw value, no key name
      outputFile: /vault/output/appc-secret
```

Install/upgrade the shared release:

```sh
helm upgrade --install vault-agent ./vault-agent \
  -f my-secrets-values.yaml
```

The `knox-secret` Secret must already exist in the release namespace with `role_id` and `secret_id` keys. Do not put Vault credentials in chart values.

## Consuming the shared secrets from another application

Applications do **not** install their own vault-agent. Instead, their chart mounts the existing PVC read-only and reads the file that corresponds to their `name` entry above:

```yaml
volumes:
  - name: vault-secrets
    persistentVolumeClaim:
      claimName: vault-agent-secrets   # "<vault-agent release name>-secrets"
      readOnly: true
containers:
  - name: app
    volumeMounts:
      - name: vault-secrets
        mountPath: /vault/secrets
        readOnly: true
```

The application then reads its own file (e.g. `/vault/secrets/appa.env`) — other apps' files are on the same volume but each app only needs to read the file matching its `name`.