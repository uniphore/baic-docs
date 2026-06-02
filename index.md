---
title: BAIC Installation Guide
layout: default
nav_order: 1
permalink: /
---

# Deploying BAIC on a Kubernetes Cluster

This guide walks you through installing the Uniphore BAIC platform into a Kubernetes cluster you control, using the `uniphore-baic` Helm chart. By the end of this guide you will have a working BAIC deployment: a web UI you can log into and an API you can call.

You will need to perform some of these steps from your local terminal (with `kubectl` and `helm` configured against your cluster), and some from whichever console manages your Kubernetes cluster.

## How the artifacts are delivered

All charts and images are hosted in a Uniphore-owned AWS ECR. Your Uniphore account team will issue you an AWS IAM credential pair (`ECR_ACCESS_KEY` + `ECR_SECRET_ACCESS_KEY`) scoped to read-only pull access. You use those credentials to **pull** charts and images from Uniphore's ECR, then **re-publish** to a container registry you control (Harbor, Artifactory, your own AWS ECR, GCR, Azure ACR, GitLab Container Registry, etc.). The cluster pulls only from your registry — Uniphore's ECR is reachable from your workstation, not from your cluster nodes.

The exact artifacts for your release:

**Source registry (Uniphore):** `910083607482.dkr.ecr.us-west-2.amazonaws.com`

**Umbrella + service Helm charts** (15 OCI artifacts under `uniphore-charts/`):

```text
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/uniphore-baic:0.1.0
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/xforge-db:0.1.0-v7bf3eea5
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/forge-backend:0.3.1-v49331f7d
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/forge-user-management:0.3.0-vff35cc44
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/forge-scheduler:0.2.0-v3e63599f
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/forge-api-gateway:0.2.0-v9b49161f
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/x-forge-ui:0.2.0-vc5680ec1
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/forge-question-answer:0.3.0-v85aa96d0
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/evaluation-service:0.6.0-v2153d752
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/pegasus-inference-api:1.4.0-ve812e651
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/guardrails-service:0.4.0-vcbe4adaa
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/slm-fine-tuning-studio:0.3.0-vdbafed12
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/autonomous-entity-label-extraction:0.3.0-v4216764a
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/deep-crawler:0.3.0-v992450ae
oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/document-categorizer:0.3.0-v8c455a3d
```

**Container images** (14 images under `xforge/`):

```text
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/forge-backend:v-9df59a3eca
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/forge-user-management:v-e580289a8c
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/forge-scheduler:v-7f2825e51f
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/forge-api-gateway:v-c2c860f150
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/x-forge-ui:v-5568accafb
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/forge-question-answer:v-17b34f7913
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/evaluation-service:v-4f32acbf1d
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/pegasus-inference-api:v-2f329891fc
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/guardrails-service:v-b3562c3323
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/slm-fine-tuning-studio:v-44a31dd636
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/autonomous-entity-label-extraction:v-dec4a11e29
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/deep-crawler:v-a43f5e057a
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/document-categorizer:v-681bf47d44
910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/xforge-db-migrations:v-86af973b41
```

[Step 1](#step-1--mirror-images-and-charts-into-your-registry) walks through pulling, re-tagging, and pushing these to your registry.

## What you'll deploy

The `uniphore-baic` chart bundles 13 BAIC microservices into a single Helm release. Each service ships as one `Deployment` + one `ClusterIP Service`. After installation you'll have ~16 pods running in one namespace.

| # | Service | Purpose |
|---|---|---|
| 1 | forge-backend | Core BAIC backend API (BFF) |
| 2 | forge-user-management | Authentication — JWT issuance, OIDC SSO |
| 3 | forge-scheduler | Background job scheduler |
| 4 | forge-ui | Web UI (React + nginx) |
| 5 | forge-api-gateway | External API gateway |
| 6 | forge-question-answer | RAG question-answering engine |
| 7 | evaluation-service | Policy / answer-quality evaluation |
| 8 | pegasus-inference-api | LLM inference proxy (OpenAI / Anthropic / etc.) |
| 9 | guardrails-service | LLM safety + hallucination classifier |
| 10 | slm-fine-tuning-studio | Small-language-model fine-tuning runtime |
| 11 | autonomous-entity-label-extraction | Entity extraction for ingestion |
| 12 | deep-crawler | Web crawler |
| 13 | document-categorizer | Document categorization |

Default resource footprint: **~12 vCPU, ~19 GiB memory** across the 16 pods. Plan for headroom on top of this for your Kubernetes cluster's system pods.

---

# Prerequisites

Before you start, make sure your environment meets each of the requirements below.

## kubectl

The Kubernetes command-line tool, `kubectl`, lets you run commands against your Kubernetes cluster. You'll use it throughout this guide to create Secrets, inspect pod status, and tail logs.

Install locally using the [official Kubernetes guides](https://kubernetes.io/docs/tasks/tools/). Version **1.27+** is recommended.

Once installed, configure `kubectl` to talk to the cluster you'll install BAIC into:

```sh
kubectl config current-context
# Should print the context name of your target cluster.
```

## helm

Helm is the package manager for Kubernetes — `uniphore-baic` is a Helm chart, and you'll use `helm` to install, upgrade, and roll back the release. Version **3.14+** is required (3.18 recommended).

See the [Helm installation guide](https://helm.sh/docs/intro/install/) for details on how to install locally.

## Kubernetes cluster

You will install BAIC into a cluster you provision. The chart is cluster-agnostic — Amazon EKS, Google GKE, Azure AKS, and self-managed clusters (kubeadm, k3s, OpenShift) all work.

| Requirement | Minimum |
|---|---|
| Kubernetes version | 1.27 |
| Worker node count | 3 |
| Per-node capacity | 4 vCPU / 8 GiB RAM |
| Aggregate capacity | 12 vCPU / 24 GiB RAM |
| Default `StorageClass` | One must exist (`kubectl get storageclass`) — required only if you enable the in-chart Postgres/Redis subcharts, otherwise unused |
| Cluster networking | Pods must be able to reach your Postgres, Redis, and the egress allowlist below |

## PostgreSQL and Redis

You provision these yourself — the umbrella chart does **not** include them. The chart consumes their connection details from two Kubernetes Secrets you'll create in [Step 2](#step-2--create-the-prerequisite-secrets).

| Service | Min version | Used by |
|---|---|---|
| PostgreSQL | **17.4** | forge-backend, forge-scheduler, forge-user-management, forge-question-answer, evaluation-service, slm-fine-tuning-studio, deep-crawler, document-categorizer, autonomous-entity-label-extraction |
| Redis (TLS, with auth token) | **7.1.0** | forge-backend, pegasus-inference-api, slm-fine-tuning-studio |

Make sure these are reachable from your Kubernetes worker nodes' subnets.

Provide a single Postgres role (e.g. `baic`) with permission to create databases (`CREATEDB`). The umbrella chart's bundled Liquibase Job (`xforge-db`) runs **before any service Deployment** and creates the two logical databases the platform needs (`xforge`, `forge-tag`) along with all schemas, tables, and seed data — you do **not** need to pre-create databases or schemas yourself.

```sql
CREATE ROLE baic LOGIN PASSWORD '<choose-a-strong-password>' CREATEDB;
```

If you'd rather run Liquibase from your own CI / DBA tooling, set `xforge-db.enabled: false` in your values file to skip the bundled Job; the application services then start straight against your pre-migrated schemas.

## Container registry (yours)

You will mirror the BAIC charts and images into a container registry you control (Harbor, Artifactory, your own AWS ECR, GCR, Azure ACR, GitLab Container Registry, etc.). Uniphore's ECR is reachable from your workstation (via the IAM creds we issued) but **not** intended to be reachable from your cluster nodes — your cluster pulls only from your registry.

Before installation, make sure:

- Your cluster's worker nodes have network reachability to **your** registry endpoint.
- You have a Docker-registry credential (username + password / token / IAM role) on your registry that allows pulling.
- You have AWS CLI configured with the `ECR_ACCESS_KEY` / `ECR_SECRET_ACCESS_KEY` Uniphore issued, so you can pull from `910083607482.dkr.ecr.us-west-2.amazonaws.com`. Test:
   ```sh
   aws ecr get-login-password --region us-west-2 \
     | docker login --username AWS --password-stdin 910083607482.dkr.ecr.us-west-2.amazonaws.com
   # → "Login Succeeded"
   ```
- You can create a `kubernetes.io/dockerconfigjson` Secret in the install namespace using your registry's credential ([Step 1](#step-1--mirror-images-and-charts-into-your-registry)).

## Credentials issued by Uniphore

Before installation, your Uniphore account team will provide:

| Credential | Purpose |
|---|---|
| `APP_CLIENT_ID` + `APP_CLIENT_SECRET` | M2M credentials for forge-user-management |
| JWT key-pair (PEM) | Token signing key-pair for forge-user-management |
| `MASTER_KEY_TOKEN_MGMT` | Symmetric key for credential encryption inside the platform |
| `LD_SDK_KEY` | LaunchDarkly server SDK key — feature-flag evaluation for forge-backend, forge-question-answer, pegasus-inference-api |
| `ECR_ACCESS_KEY` + `ECR_SECRET_ACCESS_KEY` | AWS IAM credentials for pulling BAIC Helm charts and container images from the Uniphore-hosted ECR |
| (Optional) LLM provider API keys | OpenAI, Anthropic, Fireworks, etc. — only the providers you plan to use |
| (Optional) Tenant UUID + name | Your assigned tenant identifier, used as the `tenant.id` value in your overlay |

You'll plug these into Kubernetes Secrets in [Step 2](#step-2--create-the-prerequisite-secrets).

## Network egress allowlist

The running services need outbound network access to the hosts below. If your cluster's egress is filtered (NAT gateway, network policy, perimeter firewall), make sure these are reachable from the worker nodes before you begin:

| Host | Port | Used by | Purpose |
|---|---|---|---|
| Your container registry | 443 (or your port) | Cluster (image pull) | Service container images |
| Your Postgres host | 5432 (or your port) | All Postgres-consuming services | Database |
| Your Redis host | 6379 / 6380 | forge-backend, pegasus-inference-api | Cache / session store |
| `api.openai.com` | 443 | pegasus-inference-api | (Optional) OpenAI LLM provider |
| `api.anthropic.com` | 443 | pegasus-inference-api | (Optional) Anthropic LLM provider |
| `*.fireworks.ai` | 443 | pegasus-inference-api | (Optional) Fireworks LLM provider |
| `*.apps.astra.datastax.com` | 443 | evaluation-service, forge-question-answer | (Optional) AstraDB vector store |
| `stream.launchdarkly.com` | 443 | forge-ui, forge-backend, pegasus-inference-api, forge-question-answer, guardrails-service | LaunchDarkly SDK streaming endpoint (long-lived eventsource) |
| `clientstream.launchdarkly.com` | 443 | forge-ui | LaunchDarkly client-side SDK streaming |
| `sdk.launchdarkly.com` | 443 | Same as above | LaunchDarkly REST polling fallback |
| `events.launchdarkly.com` | 443 | Same as above | LaunchDarkly analytics events |

> **LaunchDarkly egress is REQUIRED if you populate `LD_SDK_KEY` in the `launchdarkly-uniphore` and `miscellaneous` Secrets.** Several services (`pegasus-inference-api`, `guardrails-service`, `forge-question-answer`) **fail to start** if the SDK key is set but the streaming endpoints aren't reachable — their startup blocks for ~10s on `client.is_initialized()` and then raises. **HTTP HEAD against `launchdarkly.com` is not sufficient** to confirm reachability — many NATs let HEAD through but block the long-lived streaming connection. Test with:
>
> ```sh
> kubectl run -i --rm --image=curlimages/curl ld-probe -- \
>   curl -sS -m 10 -H "Authorization: $LD_SDK_KEY" \
>   -o /dev/null -w 'HTTP=%{http_code} time=%{time_total}s\n' \
>   https://stream.launchdarkly.com/all
> ```
> A successful test returns `HTTP=200`. If it times out, your egress allowlist needs updating before the install.
>
> If your environment has no LaunchDarkly access at all, leave the SDK key empty (`--from-literal=sdkKey=''` and `--from-literal=ld_sdk_key=''`). Note that with empty keys, `pegasus-inference-api`, `guardrails-service`, and `forge-question-answer` **today** also fail to start — this is a known app-side gap being tracked separately (services should tolerate empty SDK key rather than require LaunchDarkly to be reachable).

# Step 1 — Mirror images and charts into your registry

Pull each artifact from Uniphore's ECR using the IAM credentials we issued, then push them to your own registry. Your cluster will pull only from your registry — Uniphore's ECR is a workstation-side source, not a cluster-side dependency.

1. **Log in to Uniphore's ECR from your workstation** using the IAM credentials your account team shared (in 1Password). Make sure Docker Desktop (or your Docker daemon) is running first — `docker login` fails with a `docker.sock` connection error if it isn't.

    ```sh
    export AWS_ACCESS_KEY_ID=<ECR_ACCESS_KEY-from-1Password>
    export AWS_SECRET_ACCESS_KEY=<ECR_SECRET_ACCESS_KEY-from-1Password>
    export AWS_DEFAULT_REGION=us-west-2

    # Verify the credentials resolve to the expected IAM principal
    aws sts get-caller-identity
    # Expected: Account=910083607482, Arn=arn:aws:iam::910083607482:user/ecr-pull-only

    aws ecr get-login-password --region us-west-2 \
      | docker login --username AWS --password-stdin 910083607482.dkr.ecr.us-west-2.amazonaws.com
    # → Login Succeeded

    aws ecr get-login-password --region us-west-2 \
      | helm registry login --username AWS --password-stdin 910083607482.dkr.ecr.us-west-2.amazonaws.com
    # → Login Succeeded
    ```

    The ECR login token is valid for 12 hours; if you pause and resume the mirror step you may need to re-run both `login` commands.

2. **Sanity-pull one image and one chart** to confirm the credentials and network egress work end-to-end before you kick off the bulk mirror loop. If either of these fails, fix the underlying issue (firewall, expired token, wrong key) before continuing — you don't want to discover it 12 images in.

    ```sh
    docker pull 910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge/x-forge-ui:v-5568accafb
    # → "Status: Downloaded newer image for ...x-forge-ui:v-5568accafb"

    helm pull oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts/guardrails-service \
      --version 0.4.0-vcbe4adaa
    # → "Pulled: ...uniphore-charts/guardrails-service:0.4.0-vcbe4adaa"
    # → leaves guardrails-service-0.4.0-vcbe4adaa.tgz in the current directory
    ```

    If the `docker pull` fails with a TLS handshake or connect timeout, your corporate firewall is blocking outbound to `910083607482.dkr.ecr.us-west-2.amazonaws.com` (and the `*.s3.us-west-2.amazonaws.com` host that ECR layer blobs redirect to) — sort the egress before continuing.

3. **Log in to your registry**, where you will push the mirrored artifacts. Example for a generic Docker registry:

    ```sh
    docker login registry.example.com -u '<YOUR-USERNAME>' -p '<YOUR-PASSWORD-OR-TOKEN>'
    helm registry login registry.example.com -u '<YOUR-USERNAME>' -p '<YOUR-PASSWORD-OR-TOKEN>'
    ```

    For AWS ECR / GCP Artifact Registry / Azure ACR, use the respective CLI's login command instead.

4. **Mirror the 14 container images.** Pull from Uniphore, re-tag for your registry, push:

    ```sh
    SRC="910083607482.dkr.ecr.us-west-2.amazonaws.com/xforge"
    DST="registry.example.com/baic"        # your registry — and any namespace/project you want

    IMAGES=(
      "forge-backend:v-9df59a3eca"
      "forge-user-management:v-e580289a8c"
      "forge-scheduler:v-7f2825e51f"
      "forge-api-gateway:v-c2c860f150"
      "x-forge-ui:v-5568accafb"
      "forge-question-answer:v-17b34f7913"
      "evaluation-service:v-4f32acbf1d"
      "pegasus-inference-api:v-2f329891fc"
      "guardrails-service:v-b3562c3323"
      "slm-fine-tuning-studio:v-44a31dd636"
      "autonomous-entity-label-extraction:v-dec4a11e29"
      "deep-crawler:v-a43f5e057a"
      "document-categorizer:v-681bf47d44"
      "xforge-db-migrations:v-86af973b41"
    )

    for img in "${IMAGES[@]}"; do
      docker pull "$SRC/$img"
      docker tag  "$SRC/$img" "$DST/$img"
      docker push "$DST/$img"
    done
    ```

    The image **tags** (right-hand side of the `:`) must remain unchanged — your `values.yaml` overlay ([Step 3](#step-3--author-your-valuesyaml-overlay)) references them by tag. You can structure the **path** under your registry however you like (`acme/baic/forge-backend`, `baic/forge-backend`, `forge-backend` at the root — all fine); just keep it in sync with the `__REGISTRY__` placeholder in your values overlay.

5. **Mirror the 15 Helm charts.** Pull from Uniphore's ECR via `helm pull` (which downloads `.tgz` files), then push to your registry:

    ```sh
    SRC="oci://910083607482.dkr.ecr.us-west-2.amazonaws.com/uniphore-charts"
    DST="oci://registry.example.com/baic-charts"   # your OCI-compatible registry

    CHARTS=(
      "uniphore-baic:0.1.0"
      "xforge-db:0.1.0-v7bf3eea5"
      "forge-backend:0.3.1-v49331f7d"
      "forge-user-management:0.3.0-vff35cc44"
      "forge-scheduler:0.2.0-v3e63599f"
      "forge-api-gateway:0.2.0-v9b49161f"
      "x-forge-ui:0.2.0-vc5680ec1"
      "forge-question-answer:0.3.0-v85aa96d0"
      "evaluation-service:0.6.0-v2153d752"
      "pegasus-inference-api:1.4.0-ve812e651"
      "guardrails-service:0.4.0-vcbe4adaa"
      "slm-fine-tuning-studio:0.3.0-vdbafed12"
      "autonomous-entity-label-extraction:0.3.0-v4216764a"
      "deep-crawler:0.3.0-v992450ae"
      "document-categorizer:0.3.0-v8c455a3d"
    )

    mkdir -p /tmp/baic-charts && cd /tmp/baic-charts
    for chart in "${CHARTS[@]}"; do
      name="${chart%%:*}"
      version="${chart##*:}"
      helm pull "$SRC/$name" --version "$version"
      helm push "${name}-${version}.tgz" "$DST"
    done
    ```

    If your registry is not OCI-compatible (e.g. a classical Harbor `chartmuseum`), keep the `.tgz` files locally and skip the `helm push` — you'll install directly from the local file in [Step 4](#step-4--install).

6. **Create a dedicated namespace** for the BAIC release, and record it in a shell variable for the remaining steps:

    ```sh
    NAMESPACE="baic"
    kubectl create namespace $NAMESPACE
    ```

7. **Create the in-cluster image-pull Secret** using **your** registry's credentials. The Secret **must** be named `docker-config` — every subchart references that exact name:

    ```sh
    kubectl -n $NAMESPACE create secret docker-registry docker-config \
      --docker-server=registry.example.com \
      --docker-username='<YOUR-REGISTRY-USERNAME>' \
      --docker-password='<YOUR-REGISTRY-PASSWORD-OR-TOKEN>'
    ```

    > **AWS ECR / GCP Artifact Registry / Azure ACR:** these short-lived-token registries require a refresh mechanism. Use the registry-provided CronJob (e.g. ECR `aws-ecr-credential` operator) or your cluster's IRSA / Workload Identity equivalent to keep the `docker-config` Secret fresh.

---

# Step 2 — Create the prerequisite Secrets

The umbrella chart does **not** create Kubernetes Secrets. You create them in the install namespace once, before `helm install`. If a Secret a pod references is missing or has a missing key, that pod will stay in `CreateContainerConfigError` until you fix it.

There are **15 Secrets** in total. Five are required; the rest are optional and gate specific provider integrations. If your organisation manages Secrets via External Secrets Operator, Sealed Secrets, Vault Agent Injector, or SOPS — create them through your tooling instead, just keep the names and keys identical.

> If you don't plan to use an optional integration, still create its Secret with empty string values for every key. The chart references the Secret unconditionally; only the *value* being empty disables the feature.

## Required Secrets

### `xforge-db` — PostgreSQL connection

```sh
kubectl -n $NAMESPACE create secret generic xforge-db \
  --from-literal=host='<YOUR-POSTGRES-HOST>' \
  --from-literal=port='5432' \
  --from-literal=username='<YOUR-POSTGRES-USERNAME>' \
  --from-literal=password='<YOUR-POSTGRES-PASSWORD>' \
  --from-literal=database='xforge'
```

### `xforge-redis` — Redis connection

```sh
kubectl -n $NAMESPACE create secret generic xforge-redis \
  --from-literal=host='<YOUR-REDIS-HOST>' \
  --from-literal=endpoint='<YOUR-REDIS-HOST>:6380' \
  --from-literal=auth_token='<YOUR-REDIS-AUTH-TOKEN>'
```

### `forge-user-management` — Auth keys

`APP_JWT_PRIVATE_KEY` and `APP_JWT_PUBLIC_KEY` are multi-line PEM strings, so use `--from-file` for them:

```sh
cat > /tmp/jwt_private.pem <<'PEM'
-----BEGIN PRIVATE KEY-----
<UNIPHORE-ISSUED-PRIVATE-KEY-CONTENT>
-----END PRIVATE KEY-----
PEM

cat > /tmp/jwt_public.pem <<'PEM'
-----BEGIN PUBLIC KEY-----
<UNIPHORE-ISSUED-PUBLIC-KEY-CONTENT>
-----END PUBLIC KEY-----
PEM

kubectl -n $NAMESPACE create secret generic forge-user-management \
  --from-literal=APP_CLIENT_ID='<UNIPHORE-ISSUED-APP-CLIENT-ID>' \
  --from-literal=APP_CLIENT_SECRET='<UNIPHORE-ISSUED-APP-CLIENT-SECRET>' \
  --from-file=APP_JWT_PRIVATE_KEY=/tmp/jwt_private.pem \
  --from-file=APP_JWT_PUBLIC_KEY=/tmp/jwt_public.pem \
  --from-literal=MS_CLIENT_ID='' \
  --from-literal=MS_CLIENT_SECRET=''

rm -f /tmp/jwt_private.pem /tmp/jwt_public.pem
```

### `tag-db` — Forge-tag database (slm-fine-tuning-studio)

```sh
kubectl -n $NAMESPACE create secret generic tag-db \
  --from-literal=host='<YOUR-POSTGRES-HOST>' \
  --from-literal=port='5432' \
  --from-literal=username='<YOUR-POSTGRES-USERNAME>' \
  --from-literal=password='<YOUR-POSTGRES-PASSWORD>' \
  --from-literal=database='forge-tag' \
  --from-literal=jdbc_url='jdbc:postgresql://<YOUR-POSTGRES-HOST>:5432/forge-tag'
```

### `miscellaneous` — Catch-all integrations

This Secret bundles tokens for every optional integration. Populate the keys for providers you'll use; leave the rest as empty strings.

```sh
kubectl -n $NAMESPACE create secret generic miscellaneous \
  --from-literal=MASTER_KEY_TOKEN_MGMT='<UNIPHORE-ISSUED-MASTER-KEY>' \
  --from-literal=open_api_key='<OPENAI-API-KEY-OR-EMPTY>' \
  --from-literal=ANTHROPIC_API_KEY='<ANTHROPIC-API-KEY-OR-EMPTY>' \
  --from-literal=FIREWORKS_API_KEY='<FIREWORKS-API-KEY-OR-EMPTY>' \
  --from-literal=ASTRA_DB_API_ENDPOINT='<ASTRADB-ENDPOINT-OR-EMPTY>' \
  --from-literal=ASTRA_DB_APPLICATION_TOKEN='<ASTRADB-TOKEN-OR-EMPTY>' \
  --from-literal=LANDING_AI_API_KEY='<LANDINGAI-KEY-OR-EMPTY>' \
  --from-literal=unstructured_key='<UNSTRUCTURED-KEY-OR-EMPTY>' \
  --from-literal=ld_sdk_key='<LAUNCHDARKLY-SERVER-SDK-KEY-OR-EMPTY>' \
  --from-literal=LAUNCH_DARKLY_CLIENT_SIDE_ID='<LAUNCHDARKLY-CLIENT-SIDE-ID-OR-EMPTY>' \
  --from-literal=CENTXFORGE_M2M_CLIENT_ID='' \
  --from-literal=CENTXFORGE_M2M_CLIENT_SECRET='' \
  --from-literal=airbyte_s3_bucket_access_key='' \
  --from-literal=airbyte_s3_bucket_secret_key='' \
  --from-literal=gcs_credentials_json='' \
  --from-literal=workato_api_token='' \
  --from-literal=custom_oas_api_token='' \
  --from-literal=jwks_url='' \
  --from-literal=auth_token='' \
  --from-literal=AUTONOM8_SECRET='' \
  --from-literal=A8_RUNTIME_SECRET='' \
  --from-literal=AUTONOM8_API_KEY='' \
  --from-literal=AUTONOM8_RUNTIME_API_KEY='' \
  --from-literal=DATABRICKS_VISION_MODEL_URL='' \
  --from-literal=DATABRICKS_SERVING_URL='' \
  --from-literal=DATABRICKS_API_KEY='' \
  --from-literal=REDACTION_LLM_TOKEN='' \
  --from-literal=GITHUB_TOKEN_ENCRYPTION_KEY='' \
  --from-literal=agentic_search_bucket=''
```

## Optional Secrets

Create these with empty values if you don't plan to use the feature — the pods will start but the feature will be disabled.

### `oidc-secrets` — OIDC single sign-on

```sh
kubectl -n $NAMESPACE create secret generic oidc-secrets \
  --from-literal=OIDC_ISSUER_URI='' \
  --from-literal=OIDC_CLIENT_ID='' \
  --from-literal=OIDC_CLIENT_SECRET=''
```

### `external-qna-secrets` — External Q&A provider

```sh
kubectl -n $NAMESPACE create secret generic external-qna-secrets \
  --from-literal=EXTERNAL_QNA_TOKEN_URL='' \
  --from-literal=EXTERNAL_QNA_CLIENT_ID='' \
  --from-literal=EXTERNAL_QNA_CLIENT_SECRET='' \
  --from-literal=EXTERNAL_QNA_AUDIENCE=''
```

### LLM provider Secrets (pegasus-inference-api)

```sh
kubectl -n $NAMESPACE create secret generic pegasus-openai-secret      --from-literal=apiKey=''
kubectl -n $NAMESPACE create secret generic pegasus-anthropic-secret   --from-literal=apiKey=''
kubectl -n $NAMESPACE create secret generic pegasus-snowflake-secret   --from-literal=privateKey=''
kubectl -n $NAMESPACE create secret generic amd-secret                 --from-literal=apiKey=''
kubectl -n $NAMESPACE create secret generic rackspace-secret           --from-literal=apiKey=''
kubectl -n $NAMESPACE create secret generic launchdarkly-uniphore      --from-literal=sdkKey=''
kubectl -n $NAMESPACE create secret generic fireworks-credentials \
  --from-literal=accountId='uniphore' --from-literal=apiKey=''
```

## Verify all 15 Secrets are present

```sh
kubectl -n $NAMESPACE get secret \
  docker-config xforge-db xforge-redis forge-user-management tag-db \
  miscellaneous oidc-secrets external-qna-secrets \
  pegasus-openai-secret pegasus-anthropic-secret pegasus-snowflake-secret \
  amd-secret rackspace-secret launchdarkly-uniphore fireworks-credentials
```

All 15 should show `AGE` (not `Error from server (NotFound)`). If any are missing, re-run the command for that Secret.

---

# Step 3 — Author your values.yaml overlay

Your Uniphore account team will share a sample `values.yaml` (`my-values.yaml`) that covers all 13 services with sensible defaults. Two categories of values you'll typically need to adjust:

1. **Image references** — every subchart needs to know which registry to pull from. The sample sets each subchart's `image.repository` (or `image.id`) to a placeholder like `registry.example.com/baic/forge-backend`; replace `registry.example.com/baic` with the registry prefix you pushed to in [Step 1](#step-1--mirror-images-and-charts-into-your-registry). Keep the tags exactly as they appear in the [Helm Charts / Docker Images list](#how-the-artifacts-are-delivered).
2. **Browser-facing URLs (`forge-ui.vite.*`)** — these are baked into the SPA at container startup and the **browser** has to be able to reach them. The shipped sample uses `http://localhost:<port>` so you can smoke-test the install end-to-end via `kubectl port-forward` ([Step 5.5](#step-55--smoke-test-the-ui-via-kubectl-port-forward-before-you-set-up-ingress)) before you provision Ingress. Ports the sample assumes:

   | Service | Sample values URL |
   |---|---|
   | forge-ui | `http://localhost:8080` (open in browser) |
   | forge-user-management | `http://localhost:8081` |
   | forge-backend | `http://localhost:8082` |
   | pegasus-inference-api | `http://localhost:8083/openai/v1` |
   | slm-fine-tuning-studio | `http://localhost:8084/v1` |

All **inter-service URLs** in the sample (e.g. `forge-backend → forge-question-answer`, `forge-backend → pegasus-inference-api`) use Kubernetes cluster-internal DNS — `http://<service>:<port>` — which works out of the box because every service lives in the same namespace. You do not need to change these unless you split services into separate namespaces.

After your account team hands you `my-values.yaml`, **pre-flight render the chart with your overlay** to catch typos before you install:

```sh
helm template baic oci://registry.example.com/baic-charts/uniphore-baic --version 0.1.0 \
  --namespace $NAMESPACE \
  -f my-values.yaml > /tmp/rendered.yaml

grep -c '^kind: Deployment' /tmp/rendered.yaml   # expect: 13 (or fewer if you disabled some)
```

---

# Step 4 — Install

Install the umbrella chart from the location you pushed it to in [Step 1](#step-1--mirror-images-and-charts-into-your-registry).

**If you pushed charts to an OCI-compatible registry** (your own ECR, Harbor with OCI support, Artifactory, GAR, ACR):

```sh
helm install baic oci://registry.example.com/baic-charts/uniphore-baic \
  --version 0.1.0 \
  --namespace $NAMESPACE \
  -f my-values.yaml \
  --timeout 20m \
  --atomic
```

**If you kept charts as local `.tgz` files** (Chartmuseum-only or air-gapped):

```sh
helm install baic /tmp/baic-charts/uniphore-baic-0.1.0.tgz \
  --namespace $NAMESPACE \
  -f my-values.yaml \
  --timeout 20m \
  --atomic
```

`--atomic` rolls the release back automatically if any resource fails to become ready within `--timeout`. Recommended for first-time installs.

In a separate terminal, watch the pods come up:

```sh
watch kubectl -n $NAMESPACE get pods
```

You should see all 13 Deployments scale to 1/1 within ~5–10 minutes. Pods may restart once or twice on startup as they wait for Postgres migrations and other services to come up — that's expected.

If a pod is stuck in `Pending`, `CreateContainerConfigError`, `ImagePullBackOff`, or `CrashLoopBackOff`, see the [Troubleshooting](#troubleshooting) section below.

---

# Step 5 — Verify the installation

1. Confirm the release is `deployed`:

    ```sh
    helm ls -n $NAMESPACE
    # NAME  STATUS    CHART                       APP VERSION
    # baic  deployed  uniphore-baic-<version>     <version>
    ```

2. Confirm all Deployments are ready:

    ```sh
    kubectl -n $NAMESPACE get deploy
    # All should show 1/1 (or N/N if you scaled replicas).
    ```

3. Check liveness on a few core services:

    ```sh
    kubectl -n $NAMESPACE exec deploy/forge-backend -- \
      curl -sf http://forge-user-management:8080/actuator/health/liveness
    # {"status":"UP"}

    kubectl -n $NAMESPACE exec deploy/forge-backend -- \
      curl -sf http://forge-question-answer:8080/health/liveness
    # {"status":"healthy"}

    kubectl -n $NAMESPACE exec deploy/forge-backend -- \
      curl -sf http://pegasus-inference-api/health/liveness
    # {"status":"healthy"}
    ```

If all three return 200/healthy, the cluster-internal service mesh is wired correctly.

---

# Step 5.5 — Smoke test the UI via `kubectl port-forward` (before you set up Ingress)

The sample `values.yaml` your account team shares pre-wires `forge-ui`'s browser-facing URLs (`VITE_API_BASE_URL`, `VITE_USER_MANAGEMENT_BASE_URL`, `VITE_DATA_AGENTS_BASE_URL`, `VITE_MANAGE_CONNECTOR_BASE_URL`, `VITE_PEGASUS_INFERENCE_API_BASE_URL`, `VITE_FINE_TUNING_SERVICE_BASE_URL`) to `http://localhost:<port>` addresses, so you can verify login + the basic UI shell end-to-end without standing up Ingress first.

forge-ui rebuilds `env-config.js` from these values at every pod startup (nginx envsubst), so the URLs the browser sees match whatever is in `values.yaml` at install time — no rebuild required.

1. Open five terminals on your workstation, set `KUBECONFIG` + `NAMESPACE` in each, and run one port-forward per terminal:

    ```sh
    # Terminal 1 — UI (the page you load in the browser)
    kubectl -n $NAMESPACE port-forward svc/forge-ui              8080:80

    # Terminal 2 — Auth (OIDC redirect target on click of "Sign in")
    kubectl -n $NAMESPACE port-forward svc/forge-user-management 8081:8080

    # Terminal 3 — BFF (forge-backend; called after login for app data)
    kubectl -n $NAMESPACE port-forward svc/forge-backend         8082:8080

    # Terminal 4 — LLM gateway (only needed for Q&A flows)
    kubectl -n $NAMESPACE port-forward svc/pegasus-inference-api 8083:80

    # Terminal 5 — Fine-tuning UI calls (only needed for the SLM Studio page)
    kubectl -n $NAMESPACE port-forward svc/slm-fine-tuning-studio 8084:80
    ```

    The minimum needed for **login + dashboard render** is terminals 1, 2, and 3. Start 4 and 5 only when you exercise those features.

2. Open `http://localhost:8080` in a browser.

3. Click **Sign in**. The page should redirect to forge-user-management on `localhost:8081`, prompt for credentials, and on success return you to the dashboard at `localhost:8080`.

4. From the dashboard, navigate to **Knowledge Bases** or **Conversations** to verify forge-backend (port 8082) is reachable.

5. (Optional) Ask a question against a sample KB to confirm pegasus-inference-api on port 8083.

If the page loads but login fails:

| Symptom | Likely cause | Where to look |
|---|---|---|
| Browser shows white page, devtools shows `ERR_CONNECTION_REFUSED` to `localhost:8082` | forge-backend port-forward not running | Restart terminal 3 |
| Login button redirects but page is stuck at a CORS error | forge-user-management `app.allowedOrigins` doesn't include `http://localhost:8080` | Add `http://localhost:8080` to `forge-user-management.app.allowedOrigins` in your values, `helm upgrade`, re-test |
| Login succeeds but dashboard shows "API error 401" | JWT issuer mismatch between forge-ui and forge-user-management | Verify `forge-user-management.oidcSso.oauthBaseUrl` matches `http://localhost:8081` |

When you're satisfied login + basic UI work, move on to Step 6 to put real DNS + Ingress in front of these services. Remember to flip the `forge-ui.vite.*` URLs from `http://localhost:*` back to your public hostnames before you upgrade for users.

> **CSP localhost allow-list comes from values.yaml.** The sample `values.yaml` your account team shares sets `forge-ui.csp.extraOrigins: ["http://localhost:*"]`, which the x-forge-ui chart (≥ `0.2.0-vc5680ec1`) substitutes into the nginx `Content-Security-Policy` header at container startup. No manual `kubectl exec ... sed ...` patch is needed — `helm install` (or `helm upgrade`) wires it correctly. If you later switch to Ingress (Step 6), drop or repoint `csp.extraOrigins` to your customer parent domain (e.g. `["https://*.acme.example.com"]`).

---

# Step 6 — Expose services to your users

By default, all 13 services are `ClusterIP` — they're not reachable from outside the cluster. To make BAIC accessible to your end users, you'll need to provision Ingress yourself using your cluster's preferred controller (NGINX, Traefik, an ALB/Load Balancer controller, Istio, etc.).

The four services that typically need to be public:

| Service | K8s Service port | Public hostname suggestion |
|---|---|---|
| forge-ui | 80 | `baic.example.com` |
| forge-backend | 80 | `api.baic.example.com` |
| forge-user-management | 8080 | `auth.baic.example.com` |
| forge-api-gateway | 80 | `gateway.baic.example.com` |

After your Ingress is up:

1. Point DNS records at the Ingress controller's external IP / hostname.
2. Update `forge-ui.vite.*`, `forge-user-management.ui.redirectUrl`, `forge-user-management.oidcSso.oauthBaseUrl`, and `forge-api-gateway.gateway.baseUrl` in `my-values.yaml` to match your new public hostnames.
3. Apply the change:

    ```sh
    helm upgrade baic oci://registry.example.com/baic-charts/uniphore-baic --version 0.1.0 \
      --namespace $NAMESPACE \
      -f my-values.yaml \
      --atomic
    ```

> **CSP allow-list and your hostname.** The shipped `forge-ui` image's nginx `Content-Security-Policy` header explicitly lists `*.uniphorecloud.com`, `*.amazonaws.com`, `*.uniphoredev.com`, `*.uniphorestaging.com`, `*.orby.ai`, and a handful of third-party origins (LaunchDarkly, Sentry, Pendo, Datadog browser agent, etc.). If your Ingress hostname is **also under one of those parent domains** (e.g. `baic.acme.uniphorecloud.com`), no CSP changes are needed. If it's under your own corporate domain (e.g. `baic.acme.example.com`), set `forge-ui.csp.extraOrigins` in your values.yaml to include the wildcard form — the x-forge-ui chart (≥ `0.2.0-vc5680ec1`) substitutes it into the relevant CSP directives at container startup. Example:
>
> ```yaml
> forge-ui:
>   csp:
>     extraOrigins:
>       - "https://*.acme.example.com"
> ```
>
> Then `helm upgrade`. The customer-domain origins are appended to (not replacing) the baked-in Uniphore + third-party allow-list, so nothing existing breaks.

---

# Smoke test — Run a sample RAG query

Once `forge-ui` is reachable from your browser, the simplest end-to-end check is:

1. Open `https://baic.example.com` in a browser, log in with your Uniphore-issued credentials.
2. Create a new Knowledge Base and upload a sample PDF (any document up to ~10 MB).
3. Wait until ingestion completes (status `READY`). This exercises forge-backend → autonomous-entity-label-extraction → document-categorizer → evaluation-service.
4. Ask a question against the KB. This exercises forge-backend → forge-question-answer → pegasus-inference-api → guardrails-service → forge-question-answer.

If you see a grounded answer with citations, your install is healthy end-to-end.

To exercise the same flow without the UI, use the BAIC API directly:

```sh
TOKEN=$(curl -s -X POST "https://auth.baic.example.com/auth/m2m-token" \
  -d "client_id=$APP_CLIENT_ID&client_secret=$APP_CLIENT_SECRET" \
  | jq -r .access_token)

curl -X POST "https://api.baic.example.com/v1/question-answer/ask" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kb_id":"<your-kb-uuid>","question":"What is in this document?"}'
```

---

# Day-2 operations

## Upgrading

When Uniphore publishes a new chart + image set to ECR (your account team will share the new chart versions + image tags):

1. Re-run the mirror loop from [Step 1](#step-1--mirror-images-and-charts-into-your-registry) with the new versions in the `IMAGES` and `CHARTS` arrays.
2. Update the image tags in your `my-values.yaml` overlay to match the new release.
3. Upgrade in place from the new chart version:

   ```sh
   helm upgrade baic oci://registry.example.com/baic-charts/uniphore-baic \
     --version <new-version> \
     --namespace $NAMESPACE \
     -f my-values.yaml \
     --atomic \
     --timeout 20m
   ```

Read the release notes for any required values changes before upgrading.

## Rolling back

```sh
helm history baic -n $NAMESPACE
helm rollback baic <revision> -n $NAMESPACE
```

## Uninstalling

```sh
helm uninstall baic -n $NAMESPACE
```

`helm uninstall` removes Deployments, Services, and any resources owned by the chart. The 15 prerequisite Secrets you created in Step 2 are **not** owned by the chart — clean them up manually if you want a fully empty namespace:

```sh
kubectl delete namespace $NAMESPACE
```

---

# Troubleshooting

## `ImagePullBackOff` on every pod

The `docker-config` Secret is missing, or its credentials don't grant pull on your registry, or the image tag in your values overlay doesn't exist in your registry.

```sh
kubectl -n $NAMESPACE get secret docker-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq
# auths."<your-registry-host>".auth should be set

kubectl -n $NAMESPACE describe pod <pod-name> | grep -E 'manifest unknown|denied|unauthorized'
```

Fix whichever applies (re-push the missing image, update the tag in your overlay, or recreate the Secret with valid credentials), then force the kubelet to retry the pull:

```sh
kubectl -n $NAMESPACE rollout restart deployment
```

## `CreateContainerConfigError` on a specific pod

A Secret the pod references doesn't exist, or it's missing a required key.

```sh
kubectl -n $NAMESPACE describe pod <pod-name> | grep -E "couldn't find key|secret .* not found"
```

Create the missing Secret (or add the missing key with `kubectl edit secret <name>`), then:

```sh
kubectl -n $NAMESPACE rollout restart deployment/<service-name>
```

If you have [Stakater Reloader](https://github.com/stakater/Reloader) installed cluster-wide, it'll detect the Secret change and roll the Deployment automatically.

## `CrashLoopBackOff` on `forge-backend` or `forge-user-management`

The most common cause is bad PostgreSQL credentials or unreachable Postgres host. Check the logs first:

```sh
kubectl -n $NAMESPACE logs deploy/forge-backend --tail 200 | grep -iE "postgres|connection refused|timeout"
```

Then verify network reachability from inside the pod:

```sh
kubectl -n $NAMESPACE exec deploy/forge-backend -- nc -zv <YOUR-POSTGRES-HOST> 5432
```

If `nc` fails, the issue is in your cluster networking / firewall rules, not in BAIC.

## `forge-ui` shows blank page or "Network error"

Your browser can't reach the URLs configured in `forge-ui.vite.apiBaseUrl` / `vite.userManagementBaseUrl`. Make sure your Ingress is up and those URLs resolve to it ([Step 6](#step-6--expose-services-to-your-users)).

## `helm install` fails with `secrets <name> is required`

A subchart's `{{ required }}` check failed. The error message names the path that's missing in your overlay — add it and retry.

## Pods stuck in `Pending`

```sh
kubectl -n $NAMESPACE describe pod <pod-name> | tail -20
```

Most common: insufficient cluster capacity (CPU/memory requests don't fit on any node), or a `nodeSelector` in the values overlay matches no nodes. Adjust node count, instance size, or remove the `nodeSelector` if you don't run a per-tenant node pool.
