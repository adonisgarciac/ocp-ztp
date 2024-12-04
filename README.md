# ocp-ztp based on a HUB cluster installed in AWS

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