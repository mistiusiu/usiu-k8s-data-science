# Coding Standards for Agentic Assistance

## General Directives

The FluxCD GitOps approach is the mono-repo design splitting each deployment across three folders: `base`, `overlays`, and `clusters`. Overlays can be `production`, `staging`, or `development`. For clusters, within the clusters folder, each cluster deployment has its own folder. These folders follow the same structure as the `overlays` folder.

## Helm Charts

In the `base` folder, Helm charts are deployed across three files (and a the default `kustomization.yaml` file):

- `namespace.yaml`
- `repository.yaml`
- `release.yaml`

A case in point:

````yaml
# namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: authentik
````

```yaml
# repository.yaml

apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository

metadata:
  name: authentik
  namespace: authentik

spec:
  interval: 1h
  url: https://charts.goauthentik.io
```

```yaml
# release.yaml

apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease

metadata:
  name: authentik
  namespace: authentik

spec:
  # How frequently Flux reconciles this HelmRelease.
  interval: 1h

  chart:
    spec:
      # Official Authentik Helm chart.
      chart: authentik

      # renovate: datasource=helm depName=authentik registryUrl=https://charts.goauthentik.io
      version: "2026.8.1"

      # HelmRepository containing the Authentik chart.
      sourceRef:
        kind: HelmRepository
        name: authentik
        namespace: authentik

      # How frequently Flux checks for chart updates.
      interval: 1h

  # Helm values are loaded from this ConfigMap.
  #
  # The ConfigMap must exist in the same namespace as this HelmRelease.
  valuesFrom:
    - kind: ConfigMap
      name: authentik-values
      valuesKey: values.yaml

  # Create CRDs during initial installation.
  install:
    crds: CreateReplace

  # Replace CRDs when upgrading the release.
  upgrade:
    crds: CreateReplace

  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
              name: authentik-(server|worker)
            patch: |
              # 1. Inject the baseline initContainer structure safely
              - op: add
                path: /spec/template/spec/initContainers
                value:
                  - name: direct-db-migration
                    image: "placeholder" 
                    command: ["python", "manage.py", "migrate"]
                    envFrom:
                      - secretRef:
                          name: authentik-secret
              
              # 2. Copy the active dynamic tag string over the placeholder
              - op: copy
                from: /spec/template/spec/containers/0/image
                path: /spec/template/spec/initContainers/0/image

              # 3. Explicitly inject the mandatory DB Direct connection properties
              - op: add
                path: /spec/template/spec/initContainers/0/env
                value:
                  - name: AUTHENTIK_POSTGRESQL__PORT
                    value: "25060"
                  - name: AUTHENTIK_POSTGRESQL__NAME
                    value: "authentik"
                  - name: AUTHENTIK_POSTGRESQL__DISABLE_SERVER_SIDE_CURSORS
                    value: "false"
```

```yaml
# kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - repository.yaml
  - release.yaml
```

Each `overlay` folder houses 6 files (on the higher side) and a `kustomization.yaml` file:

- `certificate.yaml`
- `ingressroute.yaml`
- `middleware.yaml`
- `infisical-auth.yaml`
- `infisical-secrets.yaml`
- `values.yaml`

If an application doesn't consume any secrets the infisical files can be omitted. Moreover, if an application doesn't use any ingress routes the ingress files can be omitted.

A case in point:

```yaml
# certificate.yaml

apiVersion: cert-manager.io/v1
kind: Certificate

metadata:
  name: authentik-tls
  namespace: authentik

spec:
  secretName: authentik-tls

  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer

  dnsNames:
    - identity.afrofarmholding.com
    - identity.jcmtcportal.ac.ke
```

```yaml
# ingressroute.yaml

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: authentik
  namespace: authentik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`identity.afrofarmholding.com`) || Host(`identity.jcmtcportal.ac.ke`)
      kind: Rule
      services:
        - name: authentik-server
          port: 80
  tls:
    secretName: authentik-tls
```

```yaml
# middleware.yaml

apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-to-https
  namespace: authentik
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

```yaml
# infisical-auth.yaml

apiVersion: secrets.infisical.com/v1beta1
kind: InfisicalConnection
metadata:
  name: infisical-connection
  namespace: authentik
spec:
  address: https://infisical.afrofarmholding.com
---
apiVersion: secrets.infisical.com/v1beta1
kind: InfisicalAuth
metadata:
  name: authentik-infisical-auth
  namespace: authentik
spec:
  infisicalConnectionRef:
    name: infisical-connection
    namespace: authentik
  method: universal
  universal:
    clientIdRef:
      name: infisical-machine-identity-credentials
      key: clientId
      namespace: authentik
    clientSecretRef:
      name: infisical-machine-identity-credentials
      key: clientSecret
      namespace: authentik
```

```yaml
# infisical-secrets.yaml

apiVersion: secrets.infisical.com/v1beta1
kind: InfisicalStaticSecret
metadata:
  name: authentik-secret-static-sync
  namespace: authentik
spec:
  infisicalAuthRef:
    name: authentik-infisical-auth
    namespace: authentik

  syncOptions:
    refreshInterval: 5m

  sources:
    - projectSlug: afrofarm-infrastructure-x-an-g
      environmentSlug: prod
      secretPath: /authentik

  targets:
    - name: authentik-secret
      namespace: authentik
      kind: Secret
      creationPolicy: Owner
```

```yaml
# values.yaml

authentik:
  # Disable anonymous error reporting in production.
  error_reporting:
    enabled: false

  # SMTP configuration.
  #
  # SMTP credentials are loaded from the Infisical-generated Kubernetes Secret:
  #
  # AUTHENTIK_EMAIL__USERNAME
  # AUTHENTIK_EMAIL__PASSWORD
  #
  email:
    host: "da30.host-ww.net"
    port: 587

    # STARTTLS, normally used with port 587.
    use_tls: true

    # Do not enable this together with use_tls.
    use_ssl: false

    timeout: 30

    # This should be a verified sender address.
    from: "Sazara Identity <identity@afrofarmholding.com>"

global:
  envFrom:
    - secretRef:
        name: authentik-secret

  # Keep a few old ReplicaSets for rollback without accumulating too many.
  revisionHistoryLimit: 3

server:
  replicas: 2

  service:
    # Traefik routes traffic to this internal service.
    type: ClusterIP

  # The Traefik IngressRoute is managed as a separate Flux resource.
  ingress:
    enabled: false

  resources:
    requests:
      cpu: 250m
      memory: 512Mi

    limits:
      cpu: "1"
      memory: 1Gi

worker:
  replicas: 2

  resources:
    requests:
      cpu: 250m
      memory: 512Mi

    limits:
      cpu: "1"
      memory: 1Gi

# PostgreSQL is managed outside the Authentik Helm chart.
postgresql:
  enabled: false

# Redis is managed outside the Authentik Helm chart by the Redis Operator.
redis:
  enabled: false
```

```yaml
# kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base/authentik
  - ingressroute.yaml
  - certificate.yaml
  - middleware.yaml
  - infisical-auth.yaml
  - infisical-secrets.yaml

configMapGenerator:
  - name: authentik-values
    namespace: authentik
    files:
      - values.yaml

generatorOptions:
  disableNameSuffixHash: true

patches:
  - target:
      group: helm.toolkit.fluxcd.io
      version: v2
      kind: HelmRelease
      name: authentik
      namespace: authentik
    patch: |-
      - op: add
        path: /spec/valuesFrom
        value:
          - kind: ConfigMap
            name: authentik-values
            valuesKey: values.yaml
```

Finally, in the `clusters` overlay folder the Flux Kustomization is registered for reconciliation in the cluster. Reconciliation depends on this particular file being present.

```yaml
# authentik-ks.yaml

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization

metadata:
  name: authentik
  namespace: flux-system

spec:
  # Reconcile the Authentik infrastructure every 10 minutes.
  interval: 10m

  # Path inside the GitRepository.
  path: ./infrastructure/production/authentik

  # Remove resources that were deleted from Git.
  prune: true

  sourceRef:
    kind: GitRepository
    name: flux-system

  # Wait for required infrastructure to be ready before applying Authentik.
  dependsOn:
    - name: cert-manager
    - name: cert-manager-resources
    - name: infisical-operator
    - name: reloader
    - name: traefik

  # Maximum time Flux waits for reconciliation and health checks.
  timeout: 10m

  # Authentik resources are deployed into this namespace.
  targetNamespace: authentik
```
