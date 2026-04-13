# 🚀 Docker + CI/CD Integration

> **"Code push karo, pipeline khud kaam karega — yahi DevOps hai."**

---

## 🤔 What is CI/CD with Docker?

**CI (Continuous Integration):** Every code commit automatically builds the Docker image and runs tests.

**CD (Continuous Delivery/Deployment):** After tests pass, the image is pushed to a registry and deployed to servers — automatically.

```
Developer pushes code
        │
        ▼
   GitHub / GitLab
        │
        ▼  (triggers)
   CI/CD Pipeline
   ┌────────────────────────────────────┐
   │ 1. Checkout code                   │
   │ 2. Run linter & unit tests         │
   │ 3. Build Docker image              │
   │ 4. Scan image for vulnerabilities  │
   │ 5. Push image to registry          │
   │ 6. Deploy to staging               │
   │ 7. Run integration tests           │
   │ 8. Deploy to production            │
   └────────────────────────────────────┘
```

---

## 🐙 GitHub Actions — Complete Pipeline

Create `.github/workflows/docker.yml` in your repo:

### Simple Pipeline (Build + Push)

```yaml
name: Docker Build & Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      # 1. Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Set up Docker Buildx (for multi-platform builds)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Login to GitHub Container Registry
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # 4. Extract metadata (tags, labels)
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      # 5. Build and push
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha       # GitHub Actions cache
          cache-to: type=gha,mode=max
```

---

### Full Production Pipeline (Build + Test + Scan + Deploy)

```yaml
name: Production CI/CD Pipeline

on:
  push:
    branches: [main]
  release:
    types: [published]

env:
  IMAGE_NAME: your-dockerhub-username/my-api
  DOCKER_BUILDKIT: 1

jobs:

  # ── Job 1: Run Tests ─────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run tests in Docker
        run: |
          docker build --target builder -t test-image .
          docker run --rm test-image npm test

  # ── Job 2: Build & Scan ──────────────────────────────
  build-and-scan:
    name: Build and Security Scan
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan for vulnerabilities with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: table
          exit-code: 1           # Fail pipeline if critical CVEs found
          severity: CRITICAL,HIGH
          ignore-unfixed: true

  # ── Job 3: Push to Registry ──────────────────────────
  push:
    name: Push to Docker Hub
    runs-on: ubuntu-latest
    needs: build-and-scan
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Job 4: Deploy to Server ──────────────────────────
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: push
    environment: production     # requires manual approval in GitHub

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/my-app
            docker compose pull
            docker compose up -d --no-deps api
            docker system prune -f
            echo "Deployed: ${{ github.sha }}"
```

---

## 🦊 GitLab CI/CD Pipeline

Create `.gitlab-ci.yml` in your repo root:

```yaml
stages:
  - test
  - build
  - scan
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_BUILDKIT: "1"

# ── Test Stage ────────────────────────────────────────
test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build --target builder -t test-img .
    - docker run --rm test-img npm test

# ── Build Stage ───────────────────────────────────────
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# ── Security Scan ─────────────────────────────────────
scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_TAG
  allow_failure: false

# ── Deploy Stage ──────────────────────────────────────
deploy-production:
  stage: deploy
  image: alpine
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
  script:
    - ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST "
        cd /opt/myapp &&
        docker compose pull &&
        docker compose up -d &&
        docker system prune -f"
  only:
    - main
  environment:
    name: production
    url: https://myapp.com
  when: manual   # require manual trigger for production
```

---

## 🔑 Setting Up GitHub Secrets

In your GitHub repo: **Settings → Secrets and Variables → Actions**

```
DOCKERHUB_USERNAME    → your Docker Hub username
DOCKERHUB_TOKEN       → Docker Hub access token (not password!)
SSH_PRIVATE_KEY       → Private key for server SSH access
SERVER_HOST           → Production server IP or domain
SERVER_USER           → SSH username (ubuntu, ec2-user, etc.)
```

> **Generate Docker Hub token:** Hub → Account Settings → Security → New Access Token

---

## ⚡ Build Caching in CI/CD

Docker builds in CI are slow without caching. Use GitHub Actions cache or registry cache.

```yaml
# GitHub Actions cache (free, recommended)
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# Registry cache (useful for self-hosted runners)
- name: Build with registry cache
  uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=user/app:buildcache
    cache-to: type=registry,ref=user/app:buildcache,mode=max
```

---

## 🌊 Docker + AWS ECR + ECS Deployment

```yaml
deploy-to-ecs:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push to ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-api
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster my-cluster \
          --service my-service \
          --force-new-deployment
```

---

## ✅ CI/CD Best Practices

- **Never store secrets in code** — use CI/CD secret variables
- **Scan every build** — Trivy or Docker Scout for CVEs
- **Use build cache** — dramatically speeds up CI builds
- **Pin action versions** — `actions/checkout@v4` not `@latest`
- **Use multi-platform builds** — support both AMD64 and ARM64
- **Require manual approval** for production deployments (`environment: production`)
- **Tag images with git SHA** — always know exactly what's deployed
- **Keep pipelines fast** — if tests take 20+ minutes, developers skip them

---

**Next:** [📖 Glossary →](../16-Glossary/glossary.md)
