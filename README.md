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

## Deploying to AWS

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
