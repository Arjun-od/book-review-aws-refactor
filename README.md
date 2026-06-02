<<<<<<< HEAD
# Book Review App Deployment README

## Overview

This project is a 3-tier Book Review application deployed on AWS.

- Frontend: Next.js, served by Nginx on a frontend EC2 instance
- Backend: Node.js, Express, Sequelize, running on backend EC2 instances
- Database: MySQL on Amazon RDS
- Secrets/config: AWS Systems Manager Parameter Store
- Process manager: systemd
- Load balancing:
  - Internet-facing ALB for frontend traffic
  - Internal ALB for backend API traffic

The application is designed so the browser talks to the frontend origin, while Nginx proxies API requests to the internal backend ALB.

## Request Flow

```text
Browser
  -> Internet-facing ALB
  -> Frontend EC2 Nginx
      /       -> Next.js on 127.0.0.1:3000
      /api/   -> Internal backend ALB
  -> Backend EC2 Node.js app
  -> RDS MySQL
```

The frontend should use relative API paths:

```js
axios.get("/api/books");
```

It should not call the backend ALB directly from browser code.

## Backend Runtime Configuration

Production configuration is loaded dynamically from AWS SSM Parameter Store.

Systemd stores only SSM parameter names, not secret values.

Example backend systemd environment:

```ini
Environment=NODE_ENV=prod
Environment=PORT=5000
Environment=AWS_REGION=sa-east-1

Environment=SSM_DB_HOST_PARAM=/book-review/prod/db/host
Environment=SSM_DB_PORT_PARAM=/book-review/prod/db/port
Environment=SSM_DB_NAME_PARAM=/book-review/prod/db/name
Environment=SSM_DB_USER_PARAM=/book-review/prod/db/user
Environment=SSM_DB_PASSWORD_PARAM=/book-review/prod/db/password
Environment=SSM_ALLOWED_ORIGINS_PARAM=/book-review/prod/cors/allowed-origins
Environment=SSM_JWT_SECRET_PARAM=/book-review/prod/auth/jwt-secret
```

The actual values live in SSM. Sensitive values such as database password and JWT secret should use `SecureString`.

## Required Backend SSM Parameters

```text
/book-review/prod/db/host
/book-review/prod/db/port
/book-review/prod/db/name
/book-review/prod/db/user
/book-review/prod/db/password
/book-review/prod/cors/allowed-origins
/book-review/prod/auth/jwt-secret
```

Example CORS value:

```text
http://alb-1614644438.sa-east-1.elb.amazonaws.com
```

CORS origins must include only scheme, host, and optional port. Do not include API paths.

Correct:

```text
https://frontend.example.com
```

Wrong:

```text
https://frontend.example.com/api/users/register
```

## IAM Requirements

Backend EC2 instances should use an IAM role. Do not hardcode AWS credentials.

Required permissions:

```text
ssm:GetParameters
kms:Decrypt
```

`kms:Decrypt` is required when SecureString parameters use a customer-managed KMS key.

## Backend Startup Behavior

The backend startup flow should be:

```text
1. Load runtime config
2. Fetch SSM parameters
3. Validate JWT secret and DB config
4. Initialize Sequelize
5. Authenticate database connection
6. Register middleware and routes
7. Start Express server
```

The server should exit with status code `1` if required config or database initialization fails. This lets systemd and the ALB treat the instance as unhealthy.

## JWT Configuration

The backend must not use this in production:

```js
process.env.JWT_SECRET
```

Production JWT signing and verification should use:

```js
config.jwt.secret
```

This value is fetched from SSM using:

```ini
SSM_JWT_SECRET_PARAM=/book-review/prod/auth/jwt-secret
```

Both token creation and token verification must use the same runtime config secret:

```js
jwt.sign(payload, config.jwt.secret, { expiresIn: config.jwt.expiresIn });
jwt.verify(token, config.jwt.secret);
```

After changing JWT secret handling, users may need to log in again because old browser tokens can become invalid.

## Frontend API Configuration

The frontend should not depend on `NEXT_PUBLIC_API_URL` for production API calls.

Use a shared Axios client with same-origin base URL:

```js
const api = axios.create({
  baseURL: "/api",
});
```

This makes the browser call:

```text
https://frontend-domain.example.com/api/books
```

Nginx then forwards `/api/` to the internal backend ALB.

## Nginx Frontend Proxy

Example Nginx layout:

```nginx
location /api/ {
    proxy_pass http://internal-backend-alb.sa-east-1.elb.amazonaws.com;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Use this form when backend routes already include `/api/...`:

```nginx
proxy_pass http://internal-backend-alb.sa-east-1.elb.amazonaws.com;
```

## systemd Services

Frontend and backend should run as separate non-root users.

Frontend:

```ini
[Unit]
Description=Book Review Frontend
After=network.target

[Service]
User=webapp
Group=webapp
WorkingDirectory=/srv/book-review-app/frontend
ExecStart=/usr/bin/npm start

Environment=NODE_ENV=production
Environment=PORT=3000
Environment=HOSTNAME=127.0.0.1

Restart=always
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/srv/book-review-app/frontend

[Install]
WantedBy=multi-user.target
```

Backend:

```ini
[Unit]
Description=Book Review Backend
After=network.target

[Service]
User=appuser
Group=appuser
WorkingDirectory=/srv/book-review-app/backend
ExecStart=/usr/bin/node src/server.js

Environment=NODE_ENV=prod
Environment=PORT=5000
Environment=AWS_REGION=sa-east-1

Restart=always
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/srv/book-review-app/backend

[Install]
WantedBy=multi-user.target
```

Use root/admin privileges to manage services:

```bash
sudo systemctl daemon-reload
sudo systemctl restart book-review-frontend
sudo systemctl restart app.service
```

Application users should not have sudo access to start, stop, or edit services.

## OS-Level File Ownership

Recommended ownership:

```text
/srv/book-review-app              root:root
/srv/book-review-app/frontend     webapp:webapp
/srv/book-review-app/backend      appuser:appuser
/etc/nginx                        root:root
/etc/systemd/system               root:root
```

Recommended permissions:

```text
/srv/book-review-app              0711
frontend directories              0750
frontend files                    0640
backend directories               0750
backend files                     0640
```

The `0711` permission on `/srv/book-review-app` lets service users traverse into their known directories without listing the full application root.

Nginx does not need access to app source files when it only reverse-proxies traffic.

## Process Ownership Checks

Check running processes:

```bash
ps -eo user,pid,ppid,cmd | grep -E 'nginx|node|next' | grep -v grep
```

Expected:

```text
root      nginx master process
www-data  nginx worker process
webapp    next-server
appuser   node src/server.js
```

Check listening ports:

```bash
sudo ss -ltnp
```

Check service users:

```bash
systemctl show book-review-frontend -p User -p Group -p WorkingDirectory
systemctl show app.service -p User -p Group -p WorkingDirectory
```

## Health Checks

Backend should expose an unauthenticated health endpoint:

```text
GET /health
```

The internal backend ALB should use `/health` as its target group health check path.

Frontend ALB can health check `/` or a dedicated frontend health endpoint.

## Production Database Notes

Do not run Sequelize schema alteration in production:

```js
sync({ alter: true });
```

This is acceptable for local development but risky in production and Auto Scaling environments.

Use migrations for production schema changes.

## High Availability Notes

Recommended HA setup:

```text
Internet-facing ALB
  -> Frontend Auto Scaling Group across multiple AZs
  -> Internal backend ALB
  -> Backend Auto Scaling Group across multiple AZs
  -> RDS MySQL Multi-AZ
```

systemd handles process recovery on each instance:

```ini
Restart=always
RestartSec=5
```

Auto Scaling handles instance replacement.

RDS Multi-AZ handles database availability.

## Useful Troubleshooting Commands

Backend logs:

```bash
sudo journalctl -u app.service -f
```

Frontend logs:

```bash
sudo journalctl -u book-review-frontend -f
```

Nginx logs:

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

Check who owns port 3000:

```bash
sudo ss -ltnp | grep :3000
```

Check who owns port 5000:

```bash
sudo ss -ltnp | grep :5000
```

Test frontend service user access:

```bash
sudo -u webapp bash -lc 'cd /srv/book-review-app/frontend && pwd'
```

Test backend service user access:

```bash
sudo -u appuser bash -lc 'cd /srv/book-review-app/backend && pwd'
```

## Security Principles

- Do not run app services as root.
- Do not store real secrets in systemd files.
- Store secrets in SSM Parameter Store.
- Use IAM roles instead of AWS access keys.
- Keep frontend and backend users separate.
- Keep Nginx config root-owned.
- Keep systemd service files root-owned.
- Let app users run code, not manage services.
- Use security groups to limit traffic between tiers.

=======
# book-review-aws-refactor
A refactored 3 tiers application from loading credentials locally and low security, to loading credentials dynamically, improved security and availability.
>>>>>>> 3574c17046fc8a0d1346d3a586e0f4bbd38e033e
