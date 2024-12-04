# ocp-ztp based on a HUB cluster installed in AWS

https://docs.openshift.com/container-platform/4.17/edge_computing/ztp-preparing-the-hub-cluster.html


## INSTALL ACM

```bash
oc apply -f operators/acm/1-ns-open-cluster-management.yaml
oc apply -f operators/acm/2-og-open-cluster-management.yaml
oc apply -f operators/acm/3-sub-acm-operator-subscription.yaml
```

Wait until MultiClusterHub CRD is available:

```bash
watch oc api-resources | grep multiclusterhubs
```

Then, create multicluster and install TALM:

```bash
oc apply -f operators/acm/4-multiclusterhub-multiclusterhub.yaml
oc apply -f operators/talm/sub-talm.yaml
```

## INSTALL GITOPS

```bash
oc apply -f operators/gitops/1-ns-gitops.yaml
oc apply -f operators/gitops/2-og-gitops.yaml
oc apply -f operators/gitops/3-sub-gitops.yaml
```

## Enabling the assisted service in a connected env

```bash
cat <<EOF | oc apply -f -
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
spec:
  databaseStorage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  filesystemStorage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
  imageStorage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 100Gi
EOF
```

### Enabling central infrastructure management on Amazon Web Services

```bash
export MY_DOMAIN=cluster-mwftk.mwftk.sandbox2550.opentlc.com
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: ingress-controller-with-nlb
  namespace: openshift-ingress-operator
spec:
  domain: nlb-apps.${MY_DOMAIN}
  routeSelector:
      matchLabels:
        router-type: nlb
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: External
      providerParameters:
        type: AWS
        aws:
          type: NLB
EOF
```

```bash
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p "{\"spec\": {\"domain\": \"apps.${MY_DOMAIN}\"}}"
oc patch route assisted-image-service -p '{"metadata": {"labels": {"router-type": "nlb"}}}' -n multicluster-engine
oc patch route assisted-image-service -n multicluster-engine -p '{"spec": {"host": "assisted-image-service-multicluster-engine.nlb-apps.cluster-mwftk.mwftk.sandbox2550.opentlc.com"}}'
```

## Preparing the GitOps ZTP site configuration repository

1. Create a Git repository with the directory structure

```bash
podman pull registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.17
mkdir -p ./out
podman run --log-driver=none --rm registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.17 extract /home/ztp --tar | tar x -C ./out
```

You can use the directory structure under out/argocd/example as a reference for the structure and content of your Git repository.

2. Modify out/argocd/deployment/argocd-openshift-gitops-patch.json with the right versions.

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type=merge --patch-file out/argocd/deployment/argocd-openshift-gitops-patch.json
```

3. Disable the cluster-proxy-addon feature and remove the relevant hub cluster and managed pods that are responsible for this add-on

```bash
oc patch multiclusterengines.multicluster.openshift.io multiclusterengine --type=merge --patch-file out/argocd/deployment/disable-cluster-proxy-addon.json
```

4. Modify the two ArgoCD applications, out/argocd/deployment/clusters-app.yaml and out/argocd/deployment/policies-app.yaml pointing to your repo and paths.

5. Commit the changes.

4. Apply the pipeline configuration to your hub cluster

```bash
oc apply -k out/argocd/deployment
```


