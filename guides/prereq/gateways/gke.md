# GKE

This guide shows how to deploy llm-d with
[GKE Gateway](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-gke-inference-gateway) as your inference gateway. By the end, inference requests will be forwarded by a GKE-managed `Gateway` to your model servers via the llm-d EPP.

> [!NOTE]
> This guide assumes familiarity with
> [Gateway API](https://gateway-api.sigs.k8s.io/) and llm-d.

## Prerequisites

The following steps from the [GKE Inference Gateway deployment documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/deploy-gke-inference-gateway) should be run:

1. [Verify your prerequisites](https://cloud.google.com/kubernetes-engine/docs/how-to/deploy-gke-inference-gateway#before-you-begin)
2. [Configure a proxy-only subnet](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#configure_a_proxy-only_subnet)
3. [Enable Gateway API in your cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway)
4. [Verify your cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#verify-internal)

## Step 1: Install Gateway Inference CRDs

For GKE versions `1.34.0-gke.1626000` or later, the InferencePool CRD is automatically installed. For GKE versions earlier than `1.34.0-gke.1626000` install it as follows: 

```bash
GAIE_VERSION=v1.4.0

kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api-inference-extension/config/crd?ref=${GAIE_VERSION}"
```

Verify the APIs are available:

```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
kubectl api-resources --api-group=inference.networking.k8s.io
```

## Step 2: Deploy the Gateway

The key choice for deployment is whether you want an internal or external load balancer:

### Regional External Application Load Balancer

The class name is `gke-l7-regional-external-managed`. They are accessible to the internet. Here is an example for creating one:

```bash
kubectl apply -k "./guides/recipes/gateway/gke-l7-regional-external-managed"
```

### Regional Internal Application Load Balancer

The class name is `gke-l7-rilb`. They are accessible only to workloads within your VPC. Here is an example for creating one:

```bash
kubectl apply -k "./guides/recipes/gateway/gke-l7-rilb"
```

## Step 3: Verify the Gateway

Verify the `Gateway` is programmed:

```bash
kubectl get gateway llm-d-inference-gateway
```

Expected output:

```text
NAME                      CLASS                              ADDRESS         PROGRAMMED   AGE
llm-d-inference-gateway   gke-l7-regional-external-managed   xx.xx.xx.xx     True         30s
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
kubectl delete gateway llm-d-inference-gateway
```

## Troubleshooting

### Gateway not showing `PROGRAMMED=True`

```bash
kubectl describe gateway llm-d-inference-gateway
```

Verify all prerequisites were applied, especially [enabling Gateway API in your cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway) and [configuring a proxy-only subnet](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#configure_a_proxy-only_subnet). Also make sure the cluster is running [a supported GKE version](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/deploy-gke-inference-gateway#gateway-controller-requirements).

### HTTPRoute not accepted

```bash
kubectl describe httproute ${GUIDE_NAME}
```

Verify that `parentRefs` matches the Gateway name and `backendRefs` matches the InferencePool name.

### No response from Gateway IP

```bash
kubectl get gateway llm-d-inference-gateway -o jsonpath='{.status.addresses[0].value}'
```

If the address is empty, your Gateway may still be waiting for a LoadBalancer service. Check that your cluster supports external load balancers.

### Getting `fault filter abort` response

A couple of issues may cause this:

1. The request doesn't match the routing rules setup on HTTPRoute.
2. A misconfiguration in the gateway's backend routing. When configured correctly, the HTTPRoute status should have a condition of type `Reconciled` and reason `ReconciliationSucceeded`.
