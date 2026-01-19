# Next.js Docker Deployment on AWS EC2

Containerized Next.js app deployed with Docker Compose and fronted by an Nginx reverse proxy. Includes optional CI/CD via a self-hosted GitHub Actions runner.

## Stack
- Next.js
- Docker + Docker Compose
- Nginx
- AWS EC2
- GitHub Actions (self-hosted runner)

## Prerequisites
- Node.js 20 (for local development)
- Docker + Docker Compose (for container builds)
- AWS account (for production deployment)

## Local development
```bash
npm install
npm run dev
```

Create `.env` with any required environment variables for the app.

## Docker (local)
```bash
docker compose up -d --build
```

App should be available at `http://localhost`.

## Production deployment
Detailed EC2, permissions, and GitHub Actions setup steps live in:
- `DEPLOYMENT_STEPS.md`

## Scripts
- `npm run dev` - start local dev server
- `npm run build` - build for production
- `npm run start` - run production server
