# Kubernetes for Training on Nebius AI

This Terraform stack deploys MK8s clusters for training workloads with CPU and GPU node groups.

## Features

- Primary and optional secondary region deployment.
- CPU and GPU node groups per deployed region.
- Region-specific GPU platform, preset, and InfiniBand fabric.
- Optional observability stack and SkyPilot integration.

## Prerequisites

1. Install Nebius CLI:
   ```bash
   curl -sSL https://storage.eu-north1.nebius.cloud/cli/install.sh | bash
   ```
2. Reload your shell:
   ```bash
   exec -l $SHELL
   ```
3. Configure Nebius CLI authentication:
   https://docs.nebius.com/cli/configure/
4. Install `jq`.

## Quick Start

1. Set tenant/project/region values in `environment.sh`:
   - `NEBIUS_TENANT_ID`
   - `NEBIUS_PROJECT_ID_REGION1`
   - `NEBIUS_PROJECT_ID_REGION2`
   - `NEBIUS_REGION1`
   - `NEBIUS_REGION2`
2. Load environment and generate backend + auto tfvars:
   ```bash
   source ./environment.sh
   ```
3. Initialize Terraform:
   ```bash
   terraform init -reconfigure
   ```
4. Edit `terraform.tfvars`.
5. Preview:
   ```bash
   terraform plan
   ```
6. Apply:
   ```bash
   terraform apply
   ```

## Multi-Region Model

- `region1` is always the primary region.
- `region2` is deployed only when `enable_secondary_region = true`.
- A CPU node group and one or more GPU node groups are created in each deployed region.
- GPU clusters and GPU node groups use region-specific InfiniBand fabric settings.

## Core Variables

See `variables.tf` for full list.

### GPU settings by region

```hcl
gpu_nodes_platform_primary   = "gpu-h100-sxm"
gpu_nodes_preset_primary     = "8gpu-128vcpu-1600gb"

gpu_nodes_platform_secondary = "gpu-h200-sxm"
gpu_nodes_preset_secondary   = "8gpu-128vcpu-1600gb"
```

### InfiniBand fabric by region

```hcl
infiniband_fabric_primary   = "fabric-3"
infiniband_fabric_secondary = "fabric-5"
```

`infiniband_fabric` still works as a deprecated global fallback, but prefer the region-specific variables above.

### Preemptible nodes

```hcl
cpu_nodes_preemptible = false
gpu_nodes_preemptible = false
```

Some projects/regions do not allow preemptible for specific platforms (for example `cpu-d3`). If you see policy errors, disable preemptible for that deployment.

## SSH configuration

```hcl
ssh_user_name = "ubuntu"
ssh_public_key = {
  key = "ssh-ed25519 ..."
  # or: path = "~/.ssh/id_rsa.pub"
}
```

## Connect to Cluster

Get credentials for the primary cluster (`region1`):

```bash
nebius mk8s v1 cluster get-credentials \
  --id "$(terraform output -json kube_cluster | jq -r '.region1.id')" \
  --external
```

For the secondary cluster (`region2`), if enabled:

```bash
nebius mk8s v1 cluster get-credentials \
  --id "$(terraform output -json kube_cluster | jq -r '.region2.id')" \
  --external
```


Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

## SkyPilot Integration

1. Create and activate a Python virtual environment:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```
2. Install SkyPilot with Kubernetes support:
   ```bash
   pip install "skypilot-nightly[kubernetes]"
   ```
3. Deploy infrastructure:
   ```bash
   source ./environment.sh
   terraform apply
   ```
4. Register contexts in SkyPilot:
   ```bash
   chmod +x ./setup-skypilot-k8s.sh
   ./setup-skypilot-k8s.sh
   ```

Example:

```bash
sky launch -c k8s-cpu --cloud kubernetes "echo hello from skypilot on k8s"
sky launch -c k8s-gpu --cloud kubernetes --gpus H200 "nvidia-smi"
```

### Launch Llama 7B Fine-Tuning

To capture fine-tuning metrics in MLflow, set this in `terraform.tfvars` before `terraform apply`:

```hcl
enable_mlflow_cluster = true
```

If `enable_mlflow_cluster` is `false`, MLflow tracking is not provisioned and fine-tuning metrics will not be captured in MLflow.

Export MLflow connection variables from Terraform outputs:

```bash
export MLFLOW_TRACKING_URI=$(terraform output -json mlflow_status | jq -r '.endpoints.public_endpoint // .endpoints.public // empty')
export MLFLOW_TRACKING_USERNAME=$(terraform output -raw mlflow_admin_username)
export MLFLOW_TRACKING_PASSWORD=$(terraform output -raw mlflow_admin_password)
```

```bash
export HF_TOKEN=<your_huggingface_token>
sky launch -c llama7b-ft /Users/realz/code/skypilot-demo/skypilot-multiregion/k8s-training/skypilot/llama7b_finetune.yaml \
  --env HF_TOKEN=$HF_TOKEN \
  --env MLFLOW_TRACKING_URI=$MLFLOW_TRACKING_URI \
  --env MLFLOW_TRACKING_USERNAME=$MLFLOW_TRACKING_USERNAME \
  --env MLFLOW_TRACKING_PASSWORD=$MLFLOW_TRACKING_PASSWORD
```

## Observability

By default, Nebius Observability Agent and Grafana can be enabled via:

```hcl
enable_nebius_o11y_agent = true
enable_grafana           = true
```

Grafana password:

```bash
terraform output -raw grafana_password
```

## Storage

Filestore can be enabled with:

```hcl
enable_filestore = true
```

Mounted path on nodes: `/mnt/filestore`.

## Troubleshooting

- Node groups not created: run `terraform plan` and confirm resources `nebius_mk8s_v1_node_group.cpu-only[...]` and `nebius_mk8s_v1_node_group.gpu[...]` are in plan.
- Backend/state errors after changing environment: run `source ./environment.sh` then `terraform init -reconfigure`.
- Preemptible policy errors: set `cpu_nodes_preemptible = false` / `gpu_nodes_preemptible = false`.
- `sky: command not found` during setup: activate the project environment first: `source .venv/bin/activate`.
- `Invalid token` / `system:anonymous` during `sky check`: unset stale token and rerun setup: `unset NEBIUS_IAM_TOKEN && ./setup-skypilot-k8s.sh`.
- `sky jobs` stuck in `PENDING` with `botocore EndpointConnectionError` to `*.s3.<region>.amazonaws.com`: controller is using AWS endpoint instead of Nebius Object Storage. Set these before `sky jobs launch`, then restart API server:
  ```bash
  export AWS_PROFILE=nebius
  export AWS_DEFAULT_PROFILE=nebius
  export AWS_ENDPOINT_URL_S3=https://storage.eu-north1.nebius.cloud
  export S3_ENDPOINT=https://storage.eu-north1.nebius.cloud
  sky api stop
  ```
  Relaunch job (and add `--infra nebius` when you want Nebius VMs).
