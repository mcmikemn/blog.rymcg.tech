---
title: "K3s part 10: OpenFaaS"
date: "2020-12-25T00:10:00-06:00"
tags: ['k3s']
---

```env
## Same git repo for infrastructure as in prior posts:
FLUX_INFRA_DIR=${HOME}/git/flux-infra
CLUSTER=k3s.example.com
```

## Add the flux HelmRepository

OpenFaaS is distributed as a helm chart. The easiest way to consume this, is to
add the helm repository directly to flux, and let flux handle it. HelmRepository
objects must be created in the `flux-system` namespace, as shown:

```bash
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/flux-system/helm-repositories.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: openfaas
  namespace: flux-system
spec:
  interval: 1m
  url: https://openfaas.github.io/faas-netes/
EOF
```

Update the `flux-system/kustomization.yaml` file, to include the helm
repository:

```bash
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/flux-system/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
- helm-repositories.yaml
EOF
```

## Create namespaces for OpenFaaS

Create `kustomization.yaml` to list all of the manifests:

```bash
mkdir -p ${FLUX_INFRA_DIR}/${CLUSTER}/openfaas
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/openfaas/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- configmap.yaml
- helm.release.yaml
EOF
```

Create namespaces: `openfaas` is for the system, and `openfaas-fn` is for
running functions:

```bash
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/openfaas/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openfaas
  labels:
    role: openfaas-system
    access: openfaas-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: openfaas-fn
  labels:
    role: openfaas-fn
EOF
```

## Create OpenFaaS helm release

```bash
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/openfaas/helm.release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: openfaas
  namespace: openfaas
spec:
  interval: 5m
  chart:
    spec:
      chart: openfaas
      # version: '4.0.x'
      sourceRef:
        kind: HelmRepository
        name: openfaas
        namespace: flux-system
      interval: 1m
  valuesFrom:
  - kind: ConfigMap
    name: openfaas
    valuesKey: values.yaml
EOF
```

## Create Helm values

You can refer to the [upstream chart
values.yaml](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values.yaml)
for settings.

```bash
cat <<EOF > ${FLUX_INFRA_DIR}/${CLUSTER}/openfaas/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openfaas
  namespace: openfaas
data:
  values.yaml: |
    functionNamespace: openfaas-fn
    generateBasicAuth: true
    serviceType: ClusterIP
    ingress:
      enabled: true
      hosts:
      - host: gateway.faas.${CLUSTER}
        serviceName: gateway
        servicePort: 8080
        path: /
      annotations:
        kubernetes.io/ingress.class: traefik
        traefik.ingress.kubernetes.io/router.tls.certresolver: default
EOF
```

## Commit and push manfests

```bash
git -C ${FLUX_INFRA_DIR} add ${CLUSTER}
git -C ${FLUX_INFRA_DIR} commit -m "${CLUSTER} OpenFaaS"
```

```bash
git -C ${FLUX_INFRA_DIR} push
```

## Login to OpenFaaS

Go to https://gateway.faas.k3s.example.com in your browser

The Username is `admin` - the Password can be obtained by running:

```bash
PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 -d)
echo "OpenFaaS username: admin"
echo "OpenFaaS password: ${PASSWORD}"
```

Login using `faas-cli`:

```bash
export OPENFAAS_URL=https://gateway.faas.${CLUSTER}
echo -n ${PASSWORD} | faas-cli login -g ${OPENFAAS_URL} -u admin --password-stdin
```

## Deploy test function

...