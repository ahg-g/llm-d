# Istio

This guide shows how to deploy llm-d with [Istio](https://istio.io/) as your inference gateway. By the end, inference requests will flow from an Istio-managed `Gateway` to your model servers via the llm-d EPP.

> [!NOTE]
> This guide assumes familiarity with [Gateway API](https://gateway-api.sigs.k8s.io/) and llm-d.

## Prerequisites

* A Kubernetes cluster running one of the three most recent [Kubernetes releases](https://kubernetes.io/releases/)
* [Helm](https://helm.sh/docs/intro/install/)
* [jq](https://jqlang.org/download/)

## Step 1: Install Gateway API and Gateway API Inference Extension CRDs

Install the required Gateway API and Gateway API Inference Extension CRDs:

```bash
GATEWAY_API_VERSION=v1.5.1
GAIE_VERSION=v1.4.0

kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=${GATEWAY_API_VERSION}"
kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api-inference-extension/config/crd?ref=${GAIE_VERSION}"
```

Verify the APIs are available:

```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
kubectl api-resources --api-group=inference.networking.k8s.io
```

## Step 2: Install Istio

Install Istio with inference extension support enabled:

```bash
ISTIO_VERSION=1.29.0
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
export PATH="$PWD/istio-${ISTIO_VERSION}/bin:$PATH"
istioctl install -y \
  --set values.pilot.env.ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true
```

Verify the installation:

```bash
kubectl get pods -n istio-system
```

Expected output:

```text
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

## Step 3: Deploy the Gateway

Create a `Gateway` resource. Istio watches this resource and creates an Envoy-based proxy that accepts incoming traffic.

```bash
kubectl apply -k ./guides/recipes/gateway/istio
```

Verify the Gateway is programmed:

```bash
kubectl get gateway llm-d-inference-gateway
```

Expected output:

```text
NAME                      CLASS   ADDRESS         PROGRAMMED   AGE
llm-d-inference-gateway   istio   10.xx.xx.xx     True         30s
```

Wait until `PROGRAMMED` shows `True` before proceeding.

## Step 4: Send a Request

> [!IMPORTANT]
> Before you are able to send requests, you need to:
> 1. Deploy one of the well-lit paths to create a model server deployment, `InferencePool` and an `HTTPRoute` to connect the Gateway to the `InferencePool`.
> 2. Make sure the environment variables `${MODEL_NAME}` and `${GUIDE_NAME}` are set as part of deploying the well-lit path steps.

Get the `Gateway` external address:

```bash
export IP=$(kubectl get gateway llm-d-inference-gateway -o jsonpath='{.status.addresses[0].value}')
```

Send an inference request via the managed `Gateway`:

```bash
curl -X POST http://${IP}/v1/completions \
    -H 'Content-Type: application/json' \
    -H 'X-Gateway-Base-Model-Name: '"$GUIDE_NAME"'' \
    -d '{
        "model": '\"${MODEL_NAME}\"',
        "prompt": "How are you today?"
    }' | jq
```

## Cleanup

```bash
kubectl delete httproute llm-d-route
kubectl delete gateway llm-d-inference-gateway
istioctl uninstall --purge -y
kubectl delete namespace istio-system
kubectl delete gatewayclass istio istio-remote
kubectl delete -k "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=${GATEWAY_API_VERSION}"
kubectl delete -k "https://github.com/kubernetes-sigs/gateway-api-inference-extension/config/crd?ref=${GAIE_VERSION}"
```

## Troubleshooting

### Gateway not showing `PROGRAMMED=True`

```bash
kubectl describe gateway llm-d-inference-gateway
kubectl get pods -n istio-system
kubectl logs -n istio-system deployment/istiod --tail=20
```

Verify Istio was installed with the inference extension flag enabled.

### HTTPRoute not accepted

```bash
kubectl describe httproute llm-d-route
```

Verify that `parentRefs` matches the Gateway name and `backendRefs` matches the InferencePool name.

### No response from Gateway IP

```bash
kubectl get gateway llm-d-inference-gateway -o jsonpath='{.status.addresses[0].value}'
```

If the address is empty, your Gateway may still be waiting for a LoadBalancer service. Check that your cluster supports external load balancers.

