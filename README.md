# Digital Twin on AWS

A conversational AI **Digital Twin** that represents a real person on their professional website. It runs locally during development and is deployed to AWS as a serverless application using **Lambda, API Gateway, S3, and CloudFront**.

The twin is given rich personal context (facts, summary, communication style, and a LinkedIn/résumé PDF) and answers visitors as if it were the person it represents, while remaining transparent that it is a digital twin when pressed.

---

## Architecture

```
User Browser
    │  HTTPS
    ▼
CloudFront (CDN, HTTPS, caching)
    │
    ▼
S3 Static Website  ──► Next.js frontend (static export)
    │  HTTPS API calls
    ▼
API Gateway (HTTP API, routing, CORS)
    │
    ▼
Lambda Function (FastAPI via Mangum)
    │
    ├──► AWS Bedrock       (generates responses — Amazon Nova models)
    └──► S3 Memory Bucket  (persists conversation history as JSON)
```

| Component            | Purpose                                                        |
| -------------------- | ------------------------------------------------------------- |
| **CloudFront**       | Global CDN, provides HTTPS, caches static content             |
| **S3 (frontend)**    | Hosts the static Next.js build                                |
| **API Gateway**      | Manages API routes, handles CORS                              |
| **Lambda**           | Runs the Python FastAPI backend serverlessly                  |
| **S3 (memory)**      | Stores conversation history as one JSON file per session      |
| **AWS Bedrock**      | Managed foundation models (Amazon Nova) that power replies    |

---

## Project Structure

```
digital-twin-aws/
└── twin/
    ├── backend/                  # FastAPI backend (Python 3.12)
    │   ├── server.py             # FastAPI app: /, /health, /chat, /conversation/{id}
    │   ├── lambda_handler.py     # Mangum adapter -> handler = Mangum(app)
    │   ├── context.py            # Builds the system prompt from personal data
    │   ├── resources.py          # Loads data/ files (PDF, summary, style, facts)
    │   ├── deploy.py             # Builds lambda-deployment.zip via Docker
    │   ├── requirements.txt      # Lambda runtime dependencies
    │   ├── pyproject.toml        # uv project definition
    │   └── data/
    │       ├── facts.json        # Structured facts about the person
    │       ├── summary.txt       # Free-text personal summary
    │       ├── style.txt         # Communication-style notes
    │       └── linkedin.pdf      # LinkedIn profile / résumé PDF
    ├── frontend/                 # Next.js frontend (static export)
    └── memory/                   # Local conversation storage (dev only)
```

---

## Prerequisites

- **Python 3.12** and [`uv`](https://docs.astral.sh/uv/)
- **Node.js 18+** and npm (for the frontend)
- **Docker Desktop** (used by `deploy.py` to build Linux-native dependencies)
- [**Terraform**](https://developer.hashicorp.com/terraform/install) `>= 1.0` (for the automated deployment)
- An **AWS account** with the AWS CLI configured (`aws configure` or SSO)
- **AWS Bedrock access** to the Amazon Nova models in your region (no longer requires per-model approval as of 2026 — it's quota-based; request a quota increase only if you hit limits)

---

## Local Development

### Backend

```bash
cd twin/backend
uv sync                      # install dependencies from pyproject/uv.lock
uv run uvicorn server:app --reload
```

Create `twin/backend/.env`:

```bash
DEFAULT_AWS_REGION=us-east-1                          # region used for the Bedrock client
BEDROCK_MODEL_ID=global.amazon.nova-2-lite-v1:0       # see model options below
CORS_ORIGINS=http://localhost:3000
USE_S3=false                                          # use local ../memory for storage in dev
MEMORY_DIR=../memory
```

> The backend authenticates to Bedrock with your AWS credentials, so your local
> `aws configure` profile must have Bedrock access. No API key is needed — OpenAI has
> been removed.

**Bedrock model options** (`BEDROCK_MODEL_ID`):

- `global.amazon.nova-2-lite-v1:0` — **recommended default** (cross-region inference profile; highest quotas, fewest approvals)
- `amazon.nova-micro-v1:0` — fastest and cheapest
- `amazon.nova-lite-v1:0` — balanced
- `amazon.nova-pro-v1:0` — most capable, more expensive

If a region-specific quota error appears, try a geography-prefixed id such as
`us.amazon.nova-lite-v1:0` or `eu.amazon.nova-lite-v1:0`. `us-east-1` is a safe Bedrock
region and does not have to match the region of your other services.

The API runs at `http://localhost:8000`. Useful endpoints:

- `GET /` – API info
- `GET /health` – health check
- `POST /chat` – `{ "message": "...", "session_id": "optional" }`
- `GET /conversation/{session_id}` – conversation history

### Frontend

```bash
cd twin/frontend
npm install
npm run dev
```

Open `http://localhost:3000` and chat with your twin.

---

## Deploying with Terraform (recommended)

The whole stack is defined as code under `twin/terraform/` and driven by two helper
scripts in `twin/scripts/`. This is the **recommended** path — it provisions everything
the manual guide below does (Lambda, API Gateway, memory + frontend S3 buckets, and the
CloudFront distribution), builds and uploads the frontend, and wires the environment
variables and CORS automatically.

### What's in `twin/terraform/`

| File            | Purpose                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| `main.tf`       | All resources: S3 buckets, IAM role, Lambda, API Gateway, CloudFront, optional Route 53 + ACM for a custom domain |
| `variables.tf`  | Input variables and their validation                                    |
| `outputs.tf`    | Outputs: API URL, CloudFront URL, bucket names, Lambda name             |
| `versions.tf`   | Terraform / AWS provider version constraints                            |
| `terraform.tfvars` | Default values (used for `dev`)                                      |

Resources are named `<project_name>-<environment>-*` (e.g. `twin-dev-api`), and the S3
buckets get the AWS account ID appended for global uniqueness. Each environment lives in
its own **Terraform workspace** (`dev`, `test`, `prod`), so their states never collide.

### Input variables

| Variable                   | Default                    | Description                                        |
| -------------------------- | -------------------------- | -------------------------------------------------- |
| `project_name`             | `twin`                     | Name prefix for all resources                      |
| `environment`              | `dev`                      | One of `dev`, `test`, `prod`                        |
| `bedrock_model_id`         | `amazon.nova-micro-v1:0`   | Bedrock model the Lambda uses                      |
| `lambda_timeout`           | `60`                       | Lambda timeout (seconds)                           |
| `api_throttle_burst_limit` | `10`                       | API Gateway burst throttle                         |
| `api_throttle_rate_limit`  | `5`                        | API Gateway steady-state throttle                  |
| `use_custom_domain`        | `false`                    | Attach a custom domain (needs a Route 53 zone)     |
| `root_domain`              | `""`                       | Apex domain, e.g. `example.com`                    |

### Deploy

Make sure **Docker Desktop is running** (the script builds the Lambda zip first) and your
AWS CLI is authenticated, then from `twin/`:

```bash
./scripts/deploy.sh <environment> [project_name]
# examples
./scripts/deploy.sh dev
./scripts/deploy.sh test
./scripts/deploy.sh prod
```

`deploy.sh` runs the full pipeline end to end:

1. **Builds** the Lambda package (`cd backend && uv run deploy.py` → `lambda-deployment.zip`).
2. **`terraform init`**, then creates/selects the workspace matching `<environment>`.
3. **`terraform apply`** to provision or update all infrastructure. For `prod` it also
   passes `-var-file=prod.tfvars` (see the custom-domain note below).
4. Reads the Terraform **outputs** (API URL, frontend bucket, custom domain).
5. Writes `frontend/.env.production` with `NEXT_PUBLIC_API_URL`, runs `npm install` &
   `npm run build`, and **syncs** the static export to the frontend S3 bucket.
6. Prints the CloudFront URL, the custom domain (if any), and the API Gateway URL.

Prefer to run Terraform yourself? The scripted steps map to:

```bash
cd twin/backend && uv run deploy.py          # build lambda-deployment.zip
cd ../terraform
terraform init
terraform workspace new dev || terraform workspace select dev
terraform apply -var="project_name=twin" -var="environment=dev"
terraform output                              # inspect URLs and bucket names
```

### Custom domain (optional, typically `prod`)

To serve the site from your own domain you need a **public Route 53 hosted zone** for that
domain in the same account.

#### Step 1 — Register a domain (if you don't have one)

> ⚠️ Domain registration requires billing permissions, so sign in as the **root user** for
> this step (not your IAM user).

**Option A — Register through Route 53**

1. Sign out of your IAM user and sign in to the AWS Console as the **root user**.
2. Go to **Route 53 → Registered domains → Register domain**.
3. Search for the domain you want, add it to the cart, and complete the purchase
   (typically **$12–40/year** depending on the TLD).
4. Wait for registration to finish (**5–30 minutes**).
5. Sign back in as your IAM user to continue.

**Option B — Use a domain you already own elsewhere**

Either transfer DNS to Route 53, or create a hosted zone here (Step 2) and update the
**nameservers** at your current registrar to the ones Route 53 assigns.

#### Step 2 — Create a hosted zone (if not auto-created)

Registering through Route 53 usually creates the hosted zone automatically. If not:

1. Go to **Route 53 → Hosted zones → Create hosted zone**.
2. Enter your domain name and choose **Type: Public hosted zone**.
3. Click **Create**.

#### Step 3 — Configure and deploy

The repo ships a `twin/terraform/prod.tfvars` template — edit it and replace the
placeholder `root_domain` with your actual domain:

```hcl
project_name             = "twin"
environment              = "prod"
bedrock_model_id         = "amazon.nova-lite-v1:0"  # better model for production
lambda_timeout           = 60
api_throttle_burst_limit = 20
api_throttle_rate_limit  = 10
use_custom_domain        = true
root_domain              = "yourdomain.com"          # replace with your domain
```

Terraform then requests an ACM certificate (in `us-east-1`, as CloudFront requires),
validates it via DNS, attaches `yourdomain.com` + `www.yourdomain.com` to the distribution,
and creates the Route 53 alias records. `deploy.sh prod` picks up `prod.tfvars`
automatically. Set `use_custom_domain = false` to skip the domain and just use the default
CloudFront URL.

### Destroy / teardown

To tear an environment down, from `twin/`:

```bash
./scripts/destroy.sh <environment> [project_name]
# example
./scripts/destroy.sh test
```

`destroy.sh`:

1. Selects the `<environment>` workspace (errors if it doesn't exist).
2. **Empties both S3 buckets** first — `terraform destroy` cannot delete non-empty buckets,
   so the frontend and memory buckets are emptied via `aws s3 rm --recursive`.
   > ⚠️ This permanently deletes all stored conversation history in the memory bucket.
3. Runs `terraform destroy -auto-approve` (adding `-var-file=prod.tfvars` for `prod`).

The workspace itself is kept. To remove it completely afterwards:

```bash
cd twin/terraform
terraform workspace select default
terraform workspace delete <environment>
```

---

## Deploying to AWS

> The steps below are the **manual, console-based** equivalent of the Terraform
> deployment above — useful for understanding what gets created or for a one-off setup
> without Terraform. For repeatable deployments, prefer the scripts in the previous section.

The deployment provisions a Lambda backend, an HTTP API Gateway in front of it, two S3 buckets (memory + frontend hosting), and a CloudFront distribution for HTTPS delivery.

### Part 1 — IAM Setup

Create an IAM user group (e.g. `TwinAccess`) and attach these managed policies, then add your IAM user to the group:

- `AWSLambda_FullAccess`
- `AmazonS3FullAccess`
- `AmazonAPIGatewayAdministrator`
- `CloudFrontFullAccess`
- `IAMReadOnlyAccess`
- `AmazonBedrockFullAccess` — for Bedrock AI services
- `CloudWatchFullAccess` — for metrics and dashboards

Sign in as the IAM user for the remaining steps (not the root account).

### Part 2 — Build the Lambda Package

`deploy.py` uses the official AWS Lambda Python 3.12 Docker image to install dependencies with the correct **Linux** binaries, then zips the app code and `data/` together.

```bash
# Make sure Docker Desktop is running
cd twin/backend
uv run deploy.py
```

This produces `lambda-deployment.zip`.

> #### ⚠️ Architecture must match (x86_64 vs arm64)
>
> Some dependencies (e.g. `pydantic-core`) ship a **compiled native binary** that must
> match the Lambda function's CPU architecture. A mismatch fails at import time with:
>
> ```
> Unable to import module 'lambda_handler':
> No module named 'pydantic_core._pydantic_core'
> ```
>
> The package architecture is set in `deploy.py` and **must equal** the architecture you
> select when creating the Lambda function:
>
> | Lambda architecture | `--platform` (docker) | `--platform` (pip)        |
> | ------------------- | --------------------- | ------------------------- |
> | **x86_64**          | `linux/amd64`         | `manylinux2014_x86_64`    |
> | **arm64** (Graviton)| `linux/arm64`         | `manylinux2014_aarch64`   |
>
> On Apple Silicon Macs it is easy to create an **arm64** function by accident — if so,
> either set both flags in `deploy.py` to the arm64 values **or** recreate the function
> as x86_64. Rebuild the zip after any change.

### Part 3 — Create the Lambda Function

1. **Lambda → Create function → Author from scratch**
   - Name: `twin-api`
   - Runtime: **Python 3.12**
   - Architecture: **must match the package you built** (see warning above)
2. **Upload** `lambda-deployment.zip`.
   - Direct upload for small/fast connections, or
   - Upload to a temporary S3 bucket and point Lambda at the S3 URI (recommended for >10 MB).
3. **Runtime settings → Handler:** `lambda_handler.handler`
4. **Configuration → Environment variables:**
   - `DEFAULT_AWS_REGION` = `us-east-1` (region for the Bedrock client)
   - `BEDROCK_MODEL_ID` = `global.amazon.nova-2-lite-v1:0` (add a `us.`/`eu.` prefix if you hit a quota error)
   - `CORS_ORIGINS` = `*` (restricted later to the CloudFront URL)
   - `USE_S3` = `true`
   - `S3_BUCKET` = your memory bucket name (created next)
5. **Configuration → Permissions:** attach `AmazonBedrockFullAccess` to the Lambda **execution role** so the function can call Bedrock.
6. **Configuration → General configuration → Timeout:** `30 seconds`

**Test** with an API Gateway proxy event for `GET /health`; expect
`{"status": "healthy", "use_s3": true, "bedrock_model": "global.amazon.nova-2-lite-v1:0"}`.

### Part 4 — Create S3 Buckets

**Memory bucket** (private, e.g. `twin-memory-<suffix>`):
- Set `S3_BUCKET` in the Lambda env to this name.
- Attach `AmazonS3FullAccess` to the Lambda **execution role** (Configuration → Permissions).

**Frontend bucket** (e.g. `twin-frontend-<suffix>`):
- Uncheck "Block all public access".
- Enable **Static website hosting** (index `index.html`, error `404.html`).
- Add a public-read bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-FRONTEND-BUCKET/*"
    }
  ]
}
```

### Part 5 — API Gateway (HTTP API)

1. **Create API → HTTP API → Build**, integration type **Lambda → `twin-api`**.
2. Routes:
   - `ANY /{proxy+}`
   - `GET /`
   - `GET /health`
   - `POST /chat`
   - `OPTIONS /{proxy+}` (CORS preflight)
3. Stage: `$default` with auto-deploy enabled.
4. **CORS:** set Allow-Origin / Allow-Headers / Allow-Methods to `*` (click **Add** for each value), Max-Age `300`.
5. Copy the **Invoke URL** and verify: `https://<api-id>.execute-api.<region>.amazonaws.com/health`.

### Part 6 — Build and Deploy the Frontend

1. Point the frontend at your API. In `twin/frontend/components/twin.tsx`, replace the local URL:

   ```ts
   // from
   const response = await fetch('http://localhost:8000/chat', { ... })
   // to
   const response = await fetch('https://<api-id>.execute-api.<region>.amazonaws.com/chat', { ... })
   ```

2. Enable static export in `twin/frontend/next.config.ts`:

   ```ts
   import type { NextConfig } from "next";

   const nextConfig: NextConfig = {
     output: "export",
     images: { unoptimized: true },
   };

   export default nextConfig;
   ```

3. Build and upload:

   ```bash
   cd twin/frontend
   npm run build
   aws s3 sync out/ s3://YOUR-FRONTEND-BUCKET/ --delete
   ```

### Part 7 — CloudFront

1. **Create distribution → Pay as you go** (do **not** pick the Free plan — it can't be deleted until the billing cycle ends).
2. **Origin:** choose **Other** (not "Amazon S3"); origin domain = your S3 **website endpoint** without `http://`.
   - **Origin protocol policy: HTTP only** (S3 website hosting has no HTTPS; HTTPS here causes 504s).
3. **Default cache behavior:** Viewer protocol **Redirect HTTP to HTTPS**, methods **GET, HEAD**, cache policy **CachingOptimized**.
4. **WAF:** Do not enable (saves cost). **Price class:** North America + Europe. **Default root object:** `index.html`.
5. Wait 5–15 minutes for status **Enabled**.

### Part 8 — Lock Down CORS

Once you have the CloudFront domain, restrict the backend to it:

- Lambda → Environment variables → set `CORS_ORIGINS` to your CloudFront URL.
- It must start with `https://` and have **no trailing slash**, e.g. `https://d1234abcd.cloudfront.net`.

Then invalidate the CloudFront cache (Invalidations → path `/*`) so changes propagate.

---

## Redeploying After Code Changes

Whenever you change backend code or `requirements.txt`, rebuild the package and update the function. The CLI reads the region from your `~/.aws/config` default profile, so `--region` is optional (and `$DEFAULT_AWS_REGION` is only set if you've exported it — `.env` is not auto-loaded into your shell).

```bash
cd twin/backend
set -a; source .env; set +a          # optional: export .env vars into the shell
uv run deploy.py                      # rebuild lambda-deployment.zip

# Direct upload (fast connections):
aws lambda update-function-code \
    --function-name twin-api \
    --zip-file fileb://lambda-deployment.zip

# Or via a temporary S3 bucket (more reliable for >10 MB / slow links):
DEPLOY_BUCKET="twin-deploy-$(date +%s)"
aws s3 mb "s3://$DEPLOY_BUCKET"
aws s3 cp lambda-deployment.zip "s3://$DEPLOY_BUCKET/"
aws lambda update-function-code \
    --function-name twin-api \
    --s3-bucket "$DEPLOY_BUCKET" \
    --s3-key lambda-deployment.zip
aws s3 rm "s3://$DEPLOY_BUCKET/lambda-deployment.zip" && aws s3 rb "s3://$DEPLOY_BUCKET"
```

Wait for `"LastUpdateStatus": "Successful"` in the output, then re-run the `GET /health` test.

---

## Testing the Full Stack

1. Visit `https://<distribution>.cloudfront.net` and chat with your twin (served over HTTPS).
2. In the **memory** S3 bucket, confirm one JSON file appears per conversation session.
3. Inspect logs in **CloudWatch → Log groups → `/aws/lambda/twin-api`**.

---

## Troubleshooting

| Symptom | Likely cause / fix |
| ------- | ------------------ |
| `No module named 'pydantic_core._pydantic_core'` | Package/function **architecture mismatch** — rebuild `deploy.py` for the function's arch (see warning above). |
| **CORS errors** in the browser console | `CORS_ORIGINS` must exactly match the CloudFront URL (`https://`, no trailing `/`); ensure the `OPTIONS` route and API Gateway CORS are configured. |
| **500 Internal Server Error** | Check CloudWatch logs; verify env vars and that the execution role has S3 **and Bedrock** access. |
| **Chat not working / "Sorry, I encountered an error"** | Usually a Bedrock error (look for a 500 in the browser console). Ensure the execution role has `AmazonBedrockFullAccess`, the timeout is ≥ 30 s, and try a `us.`/`eu.` prefixed `BEDROCK_MODEL_ID` if it's a quota/region issue. |
| **`AccessDeniedException` from Bedrock** | The Lambda execution role (or your local AWS profile) lacks Bedrock access — attach `AmazonBedrockFullAccess`. |
| **Frontend not updating** | Create a CloudFront invalidation (`/*`) and clear the browser cache. |
| **Memory not persisting** | Verify `USE_S3=true`, `S3_BUCKET` is correct, and the role has `AmazonS3FullAccess`. |
| **Missing credentials in config** | Your AWS credentials are empty/unset — run `aws configure` (or `aws sso login`) so a valid `[default]` profile exists. |

---

## Cost (approximate)

- **Lambda:** first 1M requests free, then ~$0.20 / 1M
- **API Gateway (HTTP API):** first 1M free, then ~$1.00 / 1M
- **S3:** ~$0.023 / GB-month + small per-request fees
- **CloudFront:** first 1 TB free, then ~$0.085 / GB
- **Bedrock (Amazon Nova):** pay-per-token; Nova Micro/Lite are very cheap for conversational use. See [AWS Bedrock pricing](https://aws.amazon.com/bedrock/pricing/).

Typical moderate usage stays under **$5/month**. Set a CloudWatch billing alarm to be safe.

---

## License

See [LICENSE](LICENSE).
