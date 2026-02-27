# SkyPilot GTC Demo (Nebius MK8s)

This repo deploys MK8s training infrastructure with Terraform from `infra/` and provides SkyPilot examples for fine-tuning and serving.

## Prerequisites

1. Install Nebius CLI:
   ```bash
   curl -sSL https://storage.eu-north1.nebius.cloud/cli/install.sh | bash
   exec -l $SHELL
   ```
2. Configure Nebius CLI auth:
   https://docs.nebius.com/cli/configure/
3. Install required tools: `terraform`, `jq`, `yq`, `kubectl`, `uv`.
4. From repo root, work in `infra/` for Terraform:
   ```bash
   cd infra
   ```

## Terraform Deploy

1. Edit `environment.sh` values:
   - `NEBIUS_TENANT_ID`
   - `NEBIUS_PROJECT_ID_REGION1`
   - `NEBIUS_PROJECT_ID_REGION2`
   - `NEBIUS_REGION1`
   - `NEBIUS_REGION2`
2. Load environment and generate backend/auto vars:
   ```bash
   source ./environment.sh
   ```
3. Create local tfvars (ignored by git):
   ```bash
   cp terraform.tfvars.example terraform.tfvars 2>/dev/null || true
   ```
   If no example file exists, create `terraform.tfvars` manually and set your values.
4. Initialize and deploy:
   ```bash
   terraform init -reconfigure
   terraform plan
   terraform apply
   ```

## What Terraform Creates (Region 1 and Region 2)

After `terraform apply`, this stack creates:

Region 1 (`region1`, always):
- One MK8s cluster.
- Zero or one CPU node group (optional).
- One or more GPU node groups.
- One MLflow managed cluster (required for fine-tuning flow).

Region 2 (`region2`, only if `enable_secondary_region = true`):
- One MK8s cluster.
- Zero or one CPU node group (optional).
- One or more GPU node groups.

Shared behavior:
- Region-specific GPU platform and preset settings are applied.
- Node groups in each deployed region are labeled for this solution.

## Get Kube Contexts

Use the helper script from inside `infra/`:

```bash
chmod +x ./setup-skypilot-k8s.sh
./setup-skypilot-k8s.sh
```

Then verify:

```bash
terraform output -json kube_cluster \
  | jq -r 'to_entries[].value.name | "nebius-mk8s-\(.)"' \
  | while IFS= read -r ctx; do
  echo "=== $ctx ==="
  kubectl --context "$ctx" get nodes \
    -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu,CPU:.status.allocatable.cpu
done
```

You should see at least one node with `GPU` greater than `0` in at least one cluster.

## SkyPilot Setup

From `infra/`:

```bash
UV_CACHE_DIR=$PWD/.uv-cache uv venv .venv
UV_CACHE_DIR=$PWD/.uv-cache uv pip install --python .venv/bin/python "skypilot-nightly[kubernetes]==1.0.0.dev20260219"
source .venv/bin/activate
sky check
```

Sample `sky check` output:

```text
Checking credentials to enable infra for SkyPilot.
  Kubernetes: enabled [compute]
    Allowed contexts:
    ├── nebius-mk8s-skypilot-gtc-demo-2026-eu-north1: enabled.
    └── nebius-mk8s-skypilot-gtc-demo-2026-eu-west1: enabled.

🎉 Enabled infra 🎉
  Kubernetes [compute]
    Allowed contexts:
    ├── nebius-mk8s-skypilot-gtc-demo-2026-eu-north1
    └── nebius-mk8s-skypilot-gtc-demo-2026-eu-west1
```

Then confirm SkyPilot sees the enabled infra:

```bash
sky status
```

Sample `sky status` output:

```text
Enabled Infra: kubernetes/nebius-mk8s-skypilot-gtc-demo-2026-eu-north1, kubernetes/nebius-mk8s-skypilot-gtc-demo-2026-eu-west1

Clusters
NAME                           INFRA                                   RESOURCES                                STATUS  AUTOSTOP    LAUNCHED

Managed jobs
No in-progress managed jobs. (See: sky jobs -h)
```

## Fine-Tuning Run (Llama 7B)

MLflow is required for the fine-tuning flow, so set this in `terraform.tfvars` before `terraform apply`:

```hcl
enable_mlflow_cluster = true
```

Export MLflow values:

```bash
export MLFLOW_TRACKING_URI=$(terraform output -json mlflow_status | jq -r '.endpoints.public_endpoint // .endpoints.public // empty')
export MLFLOW_TRACKING_USERNAME=$(terraform output -raw mlflow_admin_username)
export MLFLOW_TRACKING_PASSWORD=$(terraform output -raw mlflow_admin_password)
```

Set HF token and launch:

```bash
export HF_TOKEN=<your_hf_token>
sky launch -c llama7b-ft skypilot/llama7b_finetune.yaml \
  --env HF_TOKEN=$HF_TOKEN \
  --secret MLFLOW_TRACKING_URI=$MLFLOW_TRACKING_URI \
  --secret MLFLOW_TRACKING_USERNAME=$MLFLOW_TRACKING_USERNAME \
  --secret MLFLOW_TRACKING_PASSWORD=$MLFLOW_TRACKING_PASSWORD
```

After the fine-tuning job starts, you can open the SkyPilot dashboard:

```bash
sky dashboard
```

Open the MLflow dashboard to track the training metrics (get the URL from Terraform output):

```bash
echo "$MLFLOW_TRACKING_URI"
echo "$MLFLOW_TRACKING_USERNAME"
echo "$MLFLOW_TRACKING_PASSWORD"
```

## Serving Run (vLLM)

If SkyPilot serve/jobs hit AWS S3 endpoints instead of Nebius, export these first:

```bash
export AWS_PROFILE=nebius
export AWS_DEFAULT_PROFILE=nebius
export AWS_ENDPOINT_URL_S3=https://storage.eu-west1.nebius.cloud
export AWS_S3_ENDPOINT_URL=https://storage.eu-west1.nebius.cloud
export S3_ENDPOINT=https://storage.eu-west1.nebius.cloud
```

Launch serve:

```bash
sky serve up -n llama7b-svc skypilot/llama7b_serve.yaml \
  --gpus H200:8 \
  --env HF_TOKEN=$HF_TOKEN \
  --env TENSOR_PARALLEL_SIZE=8
```

Optional: pin to a specific context:

```bash
sky serve up -n llama7b-svc skypilot/llama7b_serve.yaml \
  --gpus H200:8 \
  --env HF_TOKEN=$HF_TOKEN \
  --env TENSOR_PARALLEL_SIZE=8 \
  --infra kubernetes/<your-context>
```

Get the service endpoint:

```bash
sky serve status llama7b-svc --endpoint
```

Call the model (`/v1/completions`) using that endpoint:

```bash
ENDPOINT=$(sky serve status llama7b-svc --endpoint)
curl -s "$ENDPOINT/v1/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-2-7b-hf",
    "prompt": "What is distributed training?",
    "max_tokens": 128,
    "temperature": 0.7
  }'
```

## Troubleshooting

- `Invalid token` / `system:anonymous`: `unset NEBIUS_IAM_TOKEN && ./setup-skypilot-k8s.sh`.
- SkyPilot pending with S3 endpoint errors: set the S3 env vars above, then `sky api stop` and relaunch.
- Missing `sky` command: `source .venv/bin/activate`.
- Preemptible policy errors: set `cpu_nodes_preemptible = false` and/or `gpu_nodes_preemptible = false`.
