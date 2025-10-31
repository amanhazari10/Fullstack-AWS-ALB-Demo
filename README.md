# Fullstack-AWS-ALB-Demo

This repository contains a minimal full stack application (React frontend + Node.js/Express backend) and complete instructions to deploy on AWS using EC2 instances behind an Application Load Balancer (ALB). It includes Dockerfiles, infrastructure guidance, and step-by-step deployment instructions for scalability and high availability.

## Repository Structure

```
Fullstack-AWS-ALB-Demo/
├─ frontend/                # React app
│  ├─ src/
│  │  └─ App.jsx
│  ├─ public/
│  │  └─ index.html
│  ├─ package.json
│  ├─ vite.config.js
│  └─ Dockerfile
├─ backend/                 # Node.js/Express API
│  ├─ src/
│  │  └─ index.js
│  ├─ package.json
│  └─ Dockerfile
├─ infra/
│  ├─ ec2-notes.md          # EC2 provisioning and hardening notes
│  ├─ alb-setup.md          # Target group + ALB configuration
│  └─ systemd/              # Example service files (optional)
│     ├─ backend.service
│     └─ frontend.service
└─ README.md
```

## Frontend (React via Vite)

- Minimal app that fetches from the backend API at `/api/health` via an environment variable `VITE_API_BASE_URL`.
- Containerized with a multi-stage Dockerfile that builds static assets and serves via nginx.

Example files:

frontend/package.json
```
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.0"
  }
}
```

frontend/vite.config.js
```
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

frontend/src/App.jsx
```
import { useEffect, useState } from 'react'

const apiBase = import.meta.env.VITE_API_BASE_URL || ''

function App() {
  const [msg, setMsg] = useState('Loading...')
  useEffect(() => {
    fetch(`${apiBase}/api/health`).then(r => r.json()).then(d => setMsg(d.status)).catch(() => setMsg('Error'))
  }, [])
  return (
    <div style={{ fontFamily: 'sans-serif', padding: 20 }}>
      <h1>Fullstack AWS ALB Demo</h1>
      <p>Backend health: {msg}</p>
    </div>
  )
}

export default App
```

frontend/public/index.html
```
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Fullstack AWS ALB Demo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/App.jsx"></script>
  </body>
</html>
```

frontend/Dockerfile
```
# Build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* yarn.lock* pnpm-lock.yaml* ./
RUN npm ci || yarn install || pnpm install
COPY . .
RUN npm run build || yarn build || pnpm build

# Nginx serve stage
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
# Inject API base URL at runtime via envsubst (optional enhancement)
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Backend (Node.js/Express)

- Simple Express API with a health endpoint and a demo endpoint.
- Supports CORS, reads PORT from env, defaults to 3000.

backend/package.json
```
{
  "name": "backend",
  "version": "1.0.0",
  "main": "src/index.js",
  "type": "module",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.19.2"
  }
}
```

backend/src/index.js
```
import express from 'express'
import cors from 'cors'

const app = express()
app.use(cors())
app.use(express.json())

app.get('/api/health', (_req, res) => {
  res.json({ status: 'ok', ts: new Date().toISOString() })
})

app.get('/api/hello', (_req, res) => {
  res.json({ message: 'Hello from backend via ALB' })
})

const port = process.env.PORT || 3000
app.listen(port, () => console.log(`Backend listening on ${port}`))
```

backend/Dockerfile
```
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json* yarn.lock* pnpm-lock.yaml* ./
RUN npm ci || yarn install || pnpm install
COPY . .
ENV NODE_ENV=production
EXPOSE 3000
CMD ["npm", "start"]
```

## Build and Run Locally

- Frontend: `npm install && npm run dev` inside frontend
- Backend: `npm install && npm start` inside backend
- Docker (backend): `docker build -t alb-demo-backend ./backend && docker run -p 3000:3000 alb-demo-backend`
- Docker (frontend): `docker build -t alb-demo-frontend ./frontend && docker run -p 8080:80 alb-demo-frontend`

Set VITE_API_BASE_URL in frontend container to point to backend, e.g. `http://localhost:3000` when running locally.

## AWS Deployment Overview

You will deploy:
- 2+ EC2 instances for backend in private subnets (recommended) or public for simplicity.
- 1 EC2 instance (or S3+CloudFront) for frontend.
- An Application Load Balancer (ALB) in public subnets with a target group pointing to backend instances.
- Security Groups to allow 80/443 to ALB, ALB to backend port 3000, and your IP for SSH.

### VPC and Subnets
- Use default VPC or create a new VPC with 2 public subnets and 2 private subnets across 2 AZs.
- ALB must be in at least two subnets.

### Security Groups
- ALB SG: Inbound 80/443 from 0.0.0.0/0, Outbound to backend SG on 3000.
- Backend SG: Inbound 3000 from ALB SG, Outbound 0.0.0.0/0.
- SSH SG (optional): Inbound 22 from your IP.

### EC2 Backend Instances
- Amazon Linux 2023 or Ubuntu 22.04.
- Install Docker or run Node directly.
- User data example (Docker):
```
#!/bin/bash
set -e
amazon-linux-extras install -y docker || dnf install -y docker || apt-get update && apt-get install -y docker.io
systemctl enable docker && systemctl start docker
# Pull your image after you push to a registry (ECR or Docker Hub)
# docker run -d --name backend -p 3000:3000 -e NODE_ENV=production <your-registry>/alb-demo-backend:latest
```

- If running from source, install Node 18+ and `npm start` with a systemd unit.

### Target Group
- Type: Instance or IP, Protocol: HTTP, Port: 3000
- Health check: Path `/api/health`, Success codes `200`, Interval 10s, Healthy threshold 3.
- Register both backend EC2 instances.

### Application Load Balancer
- Scheme: Internet-facing, HTTP (80) or add HTTPS (443) with ACM cert.
- Listeners:
  - 80: forward to target group.
  - 443: forward to target group (requires ACM cert and HTTPS security group rule).

### Frontend Configuration
- Set environment variable `VITE_API_BASE_URL` to the ALB DNS name, e.g. `http://my-alb-123456.ap-south-1.elb.amazonaws.com`.
- If using EC2 + nginx container for frontend, run container with:
```
docker run -d -p 80:80 -e VITE_API_BASE_URL=http://<ALB-DNS> alb-demo-frontend
```
- Alternatively host frontend on S3+CloudFront and set `VITE_API_BASE_URL` at build time.

### Route 53 (Optional)
- Create a hosted zone for your domain.
- Create an A/AAAA Alias record to the ALB for `api.yourdomain.com`.
- If serving frontend via a domain, create records for the frontend host as well.

## CI/CD (Optional)
- Build and push Docker images to Amazon ECR.
- Update EC2 instances via user data + pull-new-tag or use AWS CodeDeploy/CodeDeploy Agent.

## infra/ec2-notes.md
- SSH hardening, system updates, CloudWatch agent installation.
- How to create IAM Role for EC2 (e.g., SSM access, ECR pull).

## infra/alb-setup.md
- Step-by-step screenshots and commands for creating target groups and ALB, attaching listeners, and testing health checks.

## Testing
- Access frontend public URL: you should see "Fullstack AWS ALB Demo".
- Backend health via ALB: `curl http://<ALB-DNS>/api/health` should return `{"status":"ok",...}`.
- Stop one backend instance: traffic should continue to flow via the remaining healthy target.

## Clean Up
- Terminate EC2 instances, delete target group and ALB, release Elastic IPs if used, and remove Route 53 records.

## License
MIT (add a LICENSE file if needed).
