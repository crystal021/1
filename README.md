# Cloud Deployment Guide (Minimal Version)

> **Scope of this guide:** Demonstration deployment. It covers the minimum end-to-end path needed to run the project on AWS. It is **not** a production deployment guide; a production deployment would additionally require HTTPS termination, managed secrets, monitoring, backups, CI/CD, and so on.

Minimal path to move this project from a local machine to AWS:
**Run with Docker locally → run the same stack on EC2 → expose it via API Gateway**.

---

## Overall flow

```
[Local machine: Docker Compose] ──rsync──▶ [EC2: same Docker Compose]
                                                  │
                                                  ▼
                                            [Nginx :80]
                                                  ▲
[Public HTTPS] ──▶ [AWS API Gateway: /prod/*] ────┘
```

Three high-level steps: **prepare EC2 → sync code → expose via API Gateway**.

---

## Step 1: AWS account + EC2 instance

1. Register an AWS account and open the EC2 console.
2. **Launch instance**:
   - AMI: **Ubuntu 22.04 LTS**
   - Type: **t3.medium** (2 vCPU / 4 GB)
   - Storage: 20 GB
   - Key pair: download the `.pem` private key and keep it safe
3. Attach an **Elastic IP** so the public IP stays stable across reboots.
4. **Security group — inbound rules**:

   | Port | Source | Notes |
   |------|--------|-------|
   | 22 | Your public IP | SSH management; users with dynamic IPs can widen to `0.0.0.0/0`, but only PEM key login is allowed |
   | 80 | `0.0.0.0/0` | Nginx entry; upstream traffic from API Gateway arrives here over HTTP |
   | 5173 | `0.0.0.0/0` | **Demo only** — Vite dev server. In production this should be replaced with Nginx-served static files or an HTTPS frontend |

---

## Step 2: Install Docker on EC2

SSH in, then:

```bash
ssh -i ~/path/to/key.pem ubuntu@<elastic-ip>

sudo apt update
sudo apt install -y docker.io docker-compose-plugin nginx
sudo usermod -aG docker $USER
exit                    # log out then log back in
```

---

## Step 3: Sync code from local to EC2

```bash
cd <project-root>
export SSH_KEY=~/path/to/key.pem
export EC2_HOST=<elastic-ip>
bash scripts/deploy_ec2.sh
```

The script rsyncs `frontend/`, `src/`, `app/`, `docker-compose.yml` to EC2 and rebuilds the containers.

> **First-time ordering note:** the script ends with `docker compose up`, which **requires `.env` to already exist on EC2**, otherwise the container start step fails. Recommended first-deploy order:
> 1. Run `deploy_ec2.sh` to push the code via rsync (rebuild is fine; the failed `up` is also fine).
> 2. Create `.env` on EC2 as shown in Step 4.
> 3. Run `docker compose up -d` (or `bash scripts/deploy_ec2.sh`) again.
>
> Subsequent updates are just Step 3 (one command).

---

## Step 4: Create `.env` on EC2 and start the stack

```bash
ssh -i ~/path/to/key.pem ubuntu@<elastic-ip>
cd ~/Semantic-PII-Replacement-and-Restoration-for-Cross-Border-LLM-Processing-main
cp .env.example .env
nano .env
```

Key fields:

```env
CORS_ALLOW_ORIGINS=http://<elastic-ip>:5173

VITE_API_BASE_URL=https://<api-id>.execute-api.<region>.amazonaws.com
VITE_API_PREFIX=prod
VITE_USE_MOCK=false

MAPPING_SECRET_KEY=<fernet-key>
REMOTE_AI_API_KEY=<your-gemini-key>
```

Start and self-check:

```bash
docker compose up --build -d api redis frontend

curl -sS http://127.0.0.1:8000/health
curl -sS -I http://127.0.0.1:5173 | head -1
```

Expected:

- `/health` returns `{"status":"ok","redis":"ok"}`
- `:5173` returns `HTTP/1.1 200 OK` (Vite dev server)

---

## Step 5: Install Nginx to strip the `/prod` prefix

Put the following at `/etc/nginx/sites-available/pii-api`:

```nginx
server {
    listen 80;
    server_name _;

    location /prod/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_read_timeout 300;
    }
}
```

Enable it:

```bash
sudo ln -sf /etc/nginx/sites-available/pii-api /etc/nginx/sites-enabled/pii-api
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 6: Create an API Gateway

AWS Console → API Gateway → **Create HTTP API**:

- Integration: **HTTP proxy** → `http://<elastic-ip>/prod/{proxy}`
- Route: `ANY /{proxy+}`
- Stage: `prod`

> **Why does the Integration URL include `/prod/`?**
> The Nginx `location /prod/` block is the one that performs the prefix-stripping. If you write the integration as `http://<elastic-ip>/{proxy}` (without `/prod/`), traffic bypasses that rule and falls through to a generic catch-all — it will still reach the backend, but the "Gateway → /prod → strip prefix" chain is no longer explicit, which makes debugging and future changes harder.

After deployment you will get an invoke URL of the form:

```
https://<api-id>.execute-api.<region>.amazonaws.com/prod
```

---

## Step 7: End-to-end smoke test

```bash
export API_BASE="https://<api-id>.execute-api.<region>.amazonaws.com/prod"
curl -sS "$API_BASE/health"
```

If the response is `{"status":"ok","redis":"ok"}`, the cloud deployment is working.

Open `http://<elastic-ip>:5173` in a browser and Detect + Run should work end-to-end.

> **About the frontend entry point:** browsing directly to `:5173` is a **demo-only** path served by the Vite dev server and **does not go through API Gateway**. It is intended for demos and internal acceptance only. In a production setup, the frontend should be served as static assets through Nginx (or CloudFront/S3) over HTTPS, with `VITE_API_BASE_URL` pointing at API Gateway. In other words, **only the backend API is intended to flow through API Gateway**; the frontend is a separate, independent entry point.

---

## Routine updates

| Change | Command |
|--------|---------|
| Frontend only | `bash scripts/deploy_ec2.sh --frontend-only` |
| Backend only | `bash scripts/deploy_ec2.sh --api-only` |
| Both | `bash scripts/deploy_ec2.sh` |
| Edited `.env` | SSH in, `nano .env`, then `docker compose up -d --build` |

---

## Security notes

`.env`, the `.pem` key, and `config/secrets.json` **must never be committed to Git**. `REMOTE_AI_API_KEY` should live only in the server-side `.env` and must not appear in frontend code or public curl examples.
