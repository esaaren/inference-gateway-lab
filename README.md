
# Exports 

``` bash
gcloud config set project <project here>
gcloud config set billing/quota_project <project here >
export PROJECT_ID=$(gcloud config get project)
export REGION=us-central1
export CLUSTER_NAME=erik-inf-gateway
export HF_TOKEN=
export ZONE=us-central1-a
```

# Create proxy only subnet for IG 

```bash
gcloud compute networks subnets create proxy-only-subnet \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE \
  --region=us-central1 \
  --network=default \
  --range=192.168.0.0/24

gcloud compute networks subnets describe proxy-only-subnet --region=us-central1 --format="value(state)"
```



# Create a cluster 
``` bash
gcloud container clusters create $CLUSTER_NAME \
    --project=$PROJECT_ID \
    --location=$REGION \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --release-channel=rapid \
    --num-nodes=1 \
    --enable-managed-prometheus \
    --monitoring=SYSTEM,DCGM \
    --gateway-api=standard \
    --cluster-ipv4-cidr="/21"
```

# Nodepool 

```bash
gcloud container node-pools create gpupool \
    --accelerator type=nvidia-h100-80gb,count=2,gpu-driver-version=latest \
    --project=$PROJECT_ID \
    --location=$REGION \
    --node-locations=$REGION-a \
    --cluster=$CLUSTER_NAME \
    --machine-type=a3-highgpu-2g \
    --num-nodes=1 \
    --disk-type="pd-balanced" --spot
```


# Apply metrics auth
```bash
kubectl apply -f metrics-auth.yaml
```

# Apply HF secret 
```bash
gcloud container clusters get-credentials $CLUSTER_NAME \
    --location=$REGION

kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=${HF_TOKEN} \
    --dry-run=client -o yaml | kubectl apply -f -
```

# Install inferece gateway CRDS
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.0/manifests.yaml
```

# Deploy the model server 
```bash
kubectl apply -f vllm.yaml
```


# Inference pool resource 
```bash
helm install vllm-llama3-1-8b-instruct \
  --set inferencePool.modelServers.matchLabels.app=vllm-llama3.1-8b-instruct \
  --set provider.name=gke \
  --set healthCheckPolicy.create=false \
  --version v1.0.0 \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool
```

# Deploy the inference objectives, gateway and http route 
```bash
kubectl apply -f inferenceobjective.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httproute.yaml
```

# Checks
```bash
#kubectl describe httproute my-route
kubectl describe gateway inference-gateway
kubectl get inferenceobjective.inference.networking.x-k8s.io
kubectl get gateway inference-gateway
```

# Send a request, models: food-review, meta-llama/Llama-3.1-8B-Instruct

```bash
IP=$(kubectl get gateway/inference-gateway -o jsonpath='{.status.addresses[0].value}')
PORT=80
curl -i -X POST http://${IP}:${PORT}/v1/completions \
-H "Content-Type: application/json" \
-d '{
    "model": "meta-llama/Llama-3.1-8B-Instruct",
    "prompt": "I just ate a spicy tuna roll and it was incredible. Write a short review.",
    "max_tokens": 150,
    "temperature": "0.9"
}'
```
