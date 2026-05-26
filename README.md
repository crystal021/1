# Cloud Deployment Guide

Path: local Docker → same Compose stack on EC2 → expose with API Gateway.

```
[Local: Docker Compose] ──rsync──▶ [EC2: Docker Compose]
                                            │
                                            ▼
                                       [Nginx :80]
                                            ▲
[Public HTTPS] ──▶ [API Gateway: /prod/*] ──┘
```

---

## 1. Prepare EC2

EC2 console → Launch instance:

- AMI choose : Ubuntu 22.04 LTS
- Type: t3.samll (2 vCPU / 4 GB)
- Storage: 8 GB
- Key pair: keep the downloaded `.pem`

Attach an Elastic IP make the public IP stays stable.

Security group inbound:

| Port | Source | Use |
|------|--------|-----|
| 22 | Your public IP | SSH; dynamic-IP users may use `0.0.0.0/0`, PEM-only login |
| 80 | `0.0.0.0/0` | Nginx entry; API Gateway hits this |
| 5173 | `0.0.0.0/0` | Vite dev server, demo only |

---

## 2. Install Docker on EC2

```bash
ssh -i ~/path/to/key.pem ubuntu@<elastic-ip>

sudo apt update
sudo apt install -y docker.io docker-compose-plugin nginx
sudo usermod -aG docker $USER
exit
```

Please log out and log back in for the group changes to take effect.

---

## 3. rsync code to EC2

From the project root on your local machine:

```bash
export SSH_KEY=~/path/to/key.pem
export EC2_HOST=<elastic-ip>
bash scripts/deploy_ec2.sh
```

The script syncs `frontend/`, `src/`, `app/`, `docker-compose.yml`, then runs `docker compose build && up`.

The first run will fail at `up` because `.env` does not exist on EC2 yet. The code is already on the server, move to the next step.

---

## 4. Create `.env` on EC2 and start

```bash
ssh -i ~/path/to/key.pem ubuntu@<elastic-ip>
cd ~/Semantic-PII-Replacement-and-Restoration-for-Cross-Border-LLM-Processing-main
cp .env.example .env
nano .env
```

Fields to fill in:

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

`/health` should return `{"status":"ok","redis":"ok"}`; `:5173` should return `HTTP/1.1 200 OK`.

---

## 5. Nginx to strip `/prod`

`/etc/nginx/sites-available/pii-api`:

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

FastAPI routes are `/health`, `/detect_spans`, etc. — not `/prod/...`. Nginx strips the `/prod` prefix before forwarding.

---

## 6. Create API Gateway

API Gateway → Create HTTP API:

- Integration: HTTP proxy → `http://<elastic-ip>/prod/{proxy}`
- Route: `ANY /{proxy+}`
- Stage: `prod`

The integration URL must include `/prod/` so Nginx's `location /prod/` block handles the request. Without it, Gateway traffic bypasses that rule.

Once deployed, the invoke URL looks like:

```
https://<api-id>.execute-api.<region>.amazonaws.com/prod
```

---

## 7. Verify

```bash
export API_BASE="https://<api-id>.execute-api.<region>.amazonaws.com/prod"
curl -sS "$API_BASE/health"
```

Returning `{"status":"ok","redis":"ok"}` means the chain is working.

Open `http://<elastic-ip>:5173` in a browser. Detect + Run should go through end to end.

Hitting `:5173` directly is the demo path; it does not go through API Gateway. A production frontend would be Nginx-served static files behind HTTPS, with `VITE_API_BASE_URL` pointing at Gateway.

---

## Routine updates

| Change | Command |
|--------|---------|
| Frontend | `bash scripts/deploy_ec2.sh --frontend-only` |
| Backend | `bash scripts/deploy_ec2.sh --api-only` |
| Both | `bash scripts/deploy_ec2.sh` |
| `.env` edited | SSH in, `nano .env`, then `docker compose up -d --build` |

---

## Never commit

- `.env`
- `.pem`
- `config/secrets.json`

`REMOTE_AI_API_KEY` lives only in the server-side `.env`. Do not paste it into frontend code or public curl examples.
