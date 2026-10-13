# Data Science, Analytics & Engineering K8s Cluster

![Kubernetes](https://img.shields.io/badge/Kubernetes-%23326ce5.svg?style=flat-square&logo=kubernetes&logoColor=white)
![Talos Linux](https://img.shields.io/badge/Talos_Linux-%231E293B.svg?style=flat-square&logo=talos&logoColor=white)
![FluxCD](https://img.shields.io/badge/FluxCD-008080?style=flat-square&logo=flux&logoColor=white)
![Longhorn](https://img.shields.io/badge/Longhorn-FFA500?style=flat-square&logo=longhorn&logoColor=white)
![Apache Hadoop](https://img.shields.io/badge/Apache_Hadoop-FFFF00?style=flat-square&logo=apachehadoop&logoColor=black)
![Trivy](https://img.shields.io/badge/Trivy-3F51B5?style=flat-square&logo=trivy&logoColor=white)
![Git](https://img.shields.io/badge/git-%23F05033.svg?style=flat-square&logo=git&logoColor=white)
![Reloader](https://img.shields.io/badge/Reloader-FF4500?style=flat-square&logo=loop&logoColor=white)
![Infisical](https://img.shields.io/badge/Infisical-%23000000.svg?style=flat-square&logo=infinity&logoColor=FFD700)
![Mend Renovate](https://img.shields.io/badge/Mend_Renovate-%23002F6C.svg?style=flat-square&logo=renovate&logoColor=white)

An immutable, GitOps-driven Kubernetes cluster optimized for distributed data science workflows. Powered by Talos Linux, FluxCD, Longhorn storage, and Infisical secure secret orchestration.

### FluxCD Bootstrap

Verify that the cluster is ready for FluxCD bootstrap.

```bash
flux check --pre
```

If the cluster is ready, set the GitHub token with the `Administration` and `Content` read and write permissions.

```bash
export PROJECT_ID=<project_id>
```

```bash
export GITHUB_TOKEN="$(infisical secrets get GITHUB_TOKEN \
    --projectId="$PROJECT_ID" \
    --env=prod \
    --path=/fluxcd \
    --plain)"
```

```bash
flux bootstrap github \
  --owner=mistiusiu \
  --repository=usiu-k8s-data-science \
  --path=clusters/cloud \
  --kustomization-name=usiu-data-science \
  --secret-name=usiu-data-science-auth \
  --private=false \
  --personal=true \
  --token-auth \
  --timeout=15m0s \
  --components-extra=image-reflector-controller,image-automation-controller
```

As the bootstrap process occurs the live Kustomizations can be monitored.

```bash
flux get kustomizations --watch
```

Validate that the bootstrap process completed successfully.

```bash
flux get sources git
flux check
```

If bootstrapping on a cluster where Flux has already been applied use:

```bash
kubectl create secret generic github-usiu-k8s-secret \
  --namespace=flux-system \
  --from-literal=username=mistiusiu \
  --from-literal=password=$GITHUB_TOKEN
```

```bash
flux create source git usiu-data-science \
  --url=https://github.com/mistiusiu/usiu-k8s-data-science \
  --branch=main \
  --secret-ref=github-usiu-k8s-secret \
  --export > clusters/production/usiu-data-science-source.yaml
```

```bash
flux create kustomization usiu-data-science \
  --source=GitRepository/usiu-data-science \
  --path="./clusters/cloud" \
  --prune=true \
  --interval=10m \
  --export > clusters/production/usiu-data-science-ks.yaml
```

### Infisical Machine Credentials

Infisical requires that each namespace has a machine credential set up scoped specifically to that namespace for secrets to be retrieved.

```bash
export NAMESPACE=<namespace>
export PROJECT_ID=<project_id>
```

```bash
kubectl create namespace $NAMESPACE
```

```bash
kubectl create secret generic infisical-machine-identity-credentials \
  --namespace="$NAMESPACE" \
  --from-literal=clientId="$(infisical secrets get CLIENT_ID --projectId=$PROJECT_ID --env=prod --path=/machine-credentials --plain)" \
  --from-literal=clientSecret="$(infisical secrets get CLIENT_SECRET --projectId=$PROJECT_ID --env=prod --path=/machine-credentials --plain)"
```

Validate that the secret was created successfully.

```bash
kubectl get secrets -n $NAMESPACE
```
