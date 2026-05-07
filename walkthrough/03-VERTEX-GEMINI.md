# 03 — Gemini / Vertex AI credentials

**On AWS EC2 there is no GCP metadata server**, so Application Default Credentials don't auto-resolve. Use one of the options below.

**Reference:** [Model providers](https://docs.openclaw.ai/concepts/model-providers#google-vertex-antigravity-and-gemini-cli), [Vertex AI auth](https://cloud.google.com/vertex-ai/docs/authentication)

---

## Option A: Gemini API key (recommended on AWS)

**Provider:** `google`
**Auth:** `GEMINI_API_KEY` env var

Get a key from [aistudio.google.com](https://aistudio.google.com/app/apikey). Simplest path on EC2.

Add to `.env` on the VM:

```bash
nano ~/ea-openclaw/.env
```

```
GEMINI_API_KEY=<your-key>
```

Then restart (the `dcp` alias from walkthrough 02 expands to the `-f` flags):

```bash
cd ~/ea-openclaw
dcp up -d
```

Set default model (inside the VM or via Control UI):

```bash
openclaw onboard --auth-choice gemini-api-key
```

Example models:
- `google/gemini-3-pro-preview`
- `google/gemini-2.0-flash`

---

## Option B: Vertex AI via service account JSON

**Provider:** `google-vertex`
**Auth:** Service account JSON key file (uploaded to the VM)

Use this for Vertex-specific features (regional endpoints, IAM-gated models). Requires a GCP project.

The service-account JSON lives in the project's `keys/` directory (gitignored) — e.g. `keys/ea-agent-sa.json`. Upload it from there to the EC2.

### 1. Create service account on GCP (one-time, from your laptop)

Skip if you already have `keys/ea-agent-sa.json`.

```bash
gcloud iam service-accounts create ea-agent-sa \
  --display-name="ea-agent EC2"

gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
  --member="serviceAccount:ea-agent-sa@<GCP_PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

gcloud iam service-accounts keys create keys/ea-agent-sa.json \
  --iam-account=ea-agent-sa@<GCP_PROJECT_ID>.iam.gserviceaccount.com

gcloud services enable aiplatform.googleapis.com --project=<GCP_PROJECT_ID>
```

### 2. Upload key to EC2

From the project root:

```bash
scp -i keys/open-claw.pem keys/ea-agent-sa.json ubuntu@<EC2_PUBLIC_DNS>:~/ea-agent-sa.json
ssh -i keys/open-claw.pem ubuntu@<EC2_PUBLIC_DNS> 'chmod 600 ~/ea-agent-sa.json'
```

**Treat the JSON as a secret** — `keys/` is gitignored; never commit it.

### 3. Mount into the container

Edit `~/ea-openclaw/docker-compose.prod.yml` on the VM and extend the `openclaw-gateway` service with the volume + env:

```yaml
    volumes:
      - /home/ubuntu/ea-agent-sa.json:/run/secrets/gcp-sa.json:ro
    environment:
      GOOGLE_APPLICATION_CREDENTIALS: /run/secrets/gcp-sa.json
      GOOGLE_CLOUD_PROJECT: <GCP_PROJECT_ID>
      GOOGLE_CLOUD_REGION: us-central1
```

If the SA path is host-specific and you don't want to commit it to the fork, mark the file unchanged after the edit:

```bash
git update-index --skip-worktree docker-compose.prod.yml
```

Restart:

```bash
dcp up -d
```

### 4. Set default model

```bash
openclaw models set google-vertex/gemini-2.0-flash
```

Example Vertex models:
- `google-vertex/gemini-2.0-flash`
- `google-vertex/gemini-1.5-pro`
- `google-vertex/gemini-1.5-flash`

---

## Option C: Antigravity / Gemini CLI

OAuth-based plugins. Run inside the VM:

- Antigravity: `openclaw plugins enable google-antigravity-auth`
- Gemini CLI: `openclaw plugins enable google-gemini-cli-auth`

---

## Troubleshooting

| Symptom | Fix |
|--------|-----|
| 403 / permission denied (Vertex) | Add `roles/aiplatform.user` to the GCP service account |
| `Could not load default credentials` | Confirm `GOOGLE_APPLICATION_CREDENTIALS` is set inside the container and the file is mounted |
| `API not enabled` | `gcloud services enable aiplatform.googleapis.com --project=<GCP_PROJECT_ID>` |
| Gemini API 401 | Re-check `GEMINI_API_KEY` in `.env`; restart container |
