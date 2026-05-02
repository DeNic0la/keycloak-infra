# Keycloak Infra

This repo contains the Kubernetes manifests for the Keycloak deployment that
consumes the optimized image from `/home/g/repos/keycloak-image`.

The configuration is aligned with the official Keycloak production guidance:

- `https://www.keycloak.org/server/configuration-production`
- `https://www.keycloak.org/server/containers`
- `https://www.keycloak.org/server/configuration-production#_configure_keycloak_server_with_ipv4_or_ipv6`
- `https://www.keycloak.org/server/update-compatibility#_supported_update_strategies`
- `https://www.keycloak.org/server/all-provider-config`

## Current deployment model

- cert-manager issues the TLS certificate in-cluster into the `keycloak-tls`
  secret
- ingress terminates public TLS using that `keycloak-tls` secret
- Keycloak runs behind the ingress with `KC_HTTP_ENABLED=true`
- `KC_HOSTNAME=https://keycloak.denic0la.ch`
- `KC_PROXY_HEADERS=xforwarded`
- readiness, liveness, and startup probes use the management endpoints
- database connectivity uses PostgreSQL with TLS required
- pod security is tightened with `runAsNonRoot`, `RuntimeDefault` seccomp, and
  dropped Linux capabilities

## Provider configuration posture

This deployment currently relies on Keycloak's built-in default providers unless
an environment-specific override is clearly required.

Configured provider-adjacent behavior today:

- hostname handling through `KC_HOSTNAME` and `KC_HOSTNAME_STRICT`
- reverse-proxy handling through `KC_PROXY_HEADERS=xforwarded`
- health and metrics endpoints enabled
- PostgreSQL selected as the database backend

Not configured on purpose:

- custom provider JARs
- custom theme providers
- blanket SPI overrides from `all-provider-config`
- cluster-specific provider tuning for multi-node operation, because the current
  deployment runs a single replica

Rules for future provider changes:

- add provider configuration only when there is a concrete runtime need
- document the exact SPI and provider option being changed
- treat provider or theme changes as rollout-affecting and run
  `update-compatibility` before deployment
- if custom providers are introduced, bake them into the optimized image before
  `kc.sh build`

## Secret handling

- use Sealed Secrets for committed credentials
- do not keep plaintext `secret_*.yaml` files in the repo
- the image build and GitHub Actions workflow do not require repository secrets
- do not commit live TLS certificates or private keys from cert-manager into this
  repo

For TLS, commit only the cert-manager `Certificate` manifest and the ingress
reference to the target secret. The actual certificate and private key are
materialized by cert-manager inside the cluster as a Kubernetes `Secret`.

Create a sealed secret from a local plaintext secret file:

```bash
kubeseal -f mysecret.yaml -w mysealedsecret.yaml
```

Useful cluster admin command:

```bash
kubectl get secret -n argocd argocd-secret -o yaml > secret_argocd-secret.yaml
```

## Production checklist for this deployment

Implemented now:

- public TLS at the edge
- explicit public hostname with full URL
- reverse-proxy header handling
- health and metrics endpoints
- HTTP readiness probes for bootstrap-aware traffic routing
- resource requests and limits
- container and pod security context hardening

Documented but still environment-specific:

- admin console and admin API separation onto a different hostname or protected
  path at the ingress layer
- `KC_HTTP_MAX_QUEUED_REQUESTS` based on expected load
- scaling from one replica to multiple replicas when high availability is needed
- full database server certificate verification

Known current gap:

- `KC_DB_URL_PROPERTIES=?sslmode=require` encrypts database traffic but does not
  verify the server identity. The next hardening step is to provide the database
  CA through a Kubernetes secret and move to full verification.

## Hostname and certificate matching

Per Keycloak's hostname guide, the public hostname should be explicitly
configured and should match the host exposed by the reverse proxy. In this repo
that means these three places must stay aligned:

- [certificate_keycloak.yaml](/home/g/repos/keycloak-infra/certificate_keycloak.yaml:1)
  `commonName` and `dnsNames`
- [base/ingress.yaml](/home/g/repos/keycloak-infra/base/ingress.yaml:1)
  `spec.tls[].hosts[]` and `spec.rules[].host`
- [kustomization.yaml](/home/g/repos/keycloak-infra/kustomization.yaml:1)
  `KC_HOSTNAME`

The current intended public value is:

- `https://keycloak.denic0la.ch` for Keycloak
- `keycloak.denic0la.ch` for ingress/certificate host entries

Why the formats differ:

- `KC_HOSTNAME` is set as a full URL to match Keycloak's hostname guidance for a
  fixed public URL behind a reverse proxy
- ingress and cert-manager host fields contain only the DNS hostname, not the
  URL scheme

This repo should have only one cert-manager owner for `keycloak-tls`. The
explicit `Certificate` manifest is the source of truth, so the ingress no longer
contains cert-manager issuer annotations that would cause ingress-shim to create
another `Certificate` for the same secret.

## Upgrade strategy

Keycloak supports two update strategies:

- rolling update
- recreate update

This repo must assume recreate unless proven otherwise by Keycloak's
`update-compatibility` command and by the deployment topology.

Practical implications for this deployment:

- the current live namespace has a single replica, so any rollout causes user
  impact even if Keycloak marks the configuration as rolling-compatible
- zero-downtime rolling updates require at least two healthy replicas plus
  readiness-gated traffic shifting
- patch upgrades within the same `major.minor` line may be rolling-compatible,
  but that still needs to be checked explicitly
- changing features, cache settings, providers, themes, or incompatible versions
  can require a full recreate

Recommended rollout process:

1. Generate compatibility metadata from the currently deployed Keycloak version
   and configuration.
2. Run the compatibility check with the new version and configuration.
3. If the command exit code is `0` and the deployment has at least two replicas,
   a rolling update is eligible.
4. Otherwise perform a recreate update and plan downtime.

For automation, rely on the command exit code rather than parsing the metadata
format.

## IPv4 and IPv6

Keycloak documents IP stack selection through `JAVA_OPTS_APPEND`.

Add one of these to the deployment environment only if the cluster requires it:

- prefer IPv4:
  `JAVA_OPTS_APPEND=-Djava.net.preferIPv4Stack=true`
- prefer IPv6:
  `JAVA_OPTS_APPEND=-Djava.net.preferIPv4Stack=false -Djava.net.preferIPv6Addresses=true`

Leave this unset when the cluster supports both stacks and there is no reason to
force one.

## Rollout notes

- The repo pins Keycloak `26.6.1`, but the live namespace must still be synced
  by your deployment controller for that change to take effect.
- If you keep ingress TLS termination, continue to leave HTTPS key material out
  of the Keycloak container.
- If you move TLS termination into the container later, update the deployment to
  mount the certificate, private key, and database CA as runtime secrets.
