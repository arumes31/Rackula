# Dev Environment Persistence Deployment Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enable persistent storage feature in the dev environment (d.racku.la) so layouts persist across browser sessions.

**Architecture:** Update the deploy-dev.yml workflow to build both the frontend (with persistence enabled) and the API sidecar image, then deploy both containers using the `persist` profile. VPS requires a one-time data directory setup.

**Tech Stack:** GitHub Actions, Docker, Docker Compose profiles, ghcr.io container registry

**Issue:** #894

---

## Task 1: Add API Sidecar Build Job to deploy-dev.yml

**Files:**

- Modify: `.github/workflows/deploy-dev.yml`

**Step 1: Add API image build job**

Add a new job `build-api` that builds and pushes the API sidecar image. Insert after the `build-and-deploy` job definition.

```yaml
build-api:
  runs-on: ubuntu-latest
  permissions:
    contents: read
    packages: write

  steps:
    - uses: actions/checkout@8e8c483db84b4bee98b60c0593521ed34d9990e8 # v6

    - name: Log in to Container Registry
      uses: docker/login-action@5e57cd118135c172c3672efd75eb46360885c0ef # v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata for API
      id: meta
      uses: docker/metadata-action@c299e40c65443455700f0fdfc63efafe5b349051 # v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-api
        tags: |
          type=ref,event=branch

    - name: Build and push API
      uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # v6
      with:
        context: ./api
        file: ./api/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
```

**Step 2: Verify syntax**

Run: `cat .github/workflows/deploy-dev.yml | head -80`

Expected: Valid YAML with new `build-api` job

**Step 3: Commit**

```bash
git add .github/workflows/deploy-dev.yml
git commit -m "ci: add API sidecar build job to deploy-dev workflow"
```

---

## Task 2: Enable Persistence in Frontend Build

**Files:**

- Modify: `.github/workflows/deploy-dev.yml`

**Step 1: Add VITE_PERSIST_ENABLED build arg**

In the `build-and-deploy` job, add the persistence flag to build-args:

```yaml
- name: Build and push
  uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # v6
  with:
    context: .
    file: deploy/Dockerfile
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    build-args: |
      VITE_ENV=development
      VITE_PERSIST_ENABLED=true
      VITE_UMAMI_ENABLED=true
      VITE_UMAMI_SCRIPT_URL=${{ vars.UMAMI_SCRIPT_URL }}
      VITE_UMAMI_WEBSITE_ID=${{ vars.UMAMI_DEV_WEBSITE_ID }}
```

**Step 2: Commit**

```bash
git add .github/workflows/deploy-dev.yml
git commit -m "ci: enable persistence in dev frontend build"
```

---

## Task 3: Update Deploy Job to Use Persist Profile

**Files:**

- Modify: `.github/workflows/deploy-dev.yml`

**Step 1: Update deploy job dependencies and commands**

The deploy job needs to wait for both builds and use the persist profile:

```yaml
deploy:
  needs: [build-and-deploy, build-api]
  runs-on: [self-hosted, vps-rackula]
  permissions: {}

  steps:
    - name: Deploy to dev
      run: |
        cd /opt/rackula/rackula-dev
        docker compose --profile persist pull
        docker compose --profile persist up -d
```

**Step 2: Commit**

```bash
git add .github/workflows/deploy-dev.yml
git commit -m "ci: deploy with persist profile for API sidecar"
```

---

## Task 4: Update Smoke Test to Verify API

**Files:**

- Modify: `.github/workflows/deploy-dev.yml`

**Step 1: Add API health check to smoke test**

```yaml
smoke-test:
  needs: deploy
  runs-on: [self-hosted, vps-rackula]
  permissions: {}

  steps:
    - name: Verify deployment
      run: |
        sleep 5
        curl -sf --retry 5 --retry-delay 3 https://d.racku.la
        curl -sf --retry 5 --retry-delay 3 https://d.racku.la/api/layouts
```

**Step 2: Commit**

```bash
git add .github/workflows/deploy-dev.yml
git commit -m "ci: add API health check to dev smoke test"
```

---

## Task 5: Setup VPS Data Directory

**Files:**

- None (VPS configuration)

**Step 1: SSH to VPS and create data directory**

```bash
ssh vps-web-rackula "sudo mkdir -p /opt/rackula/rackula-dev/data && sudo chown 1001:1001 /opt/rackula/rackula-dev/data"
```

Expected: Directory created with correct ownership

**Step 2: Verify directory exists**

```bash
ssh vps-web-rackula "ls -la /opt/rackula/rackula-dev/data"
```

Expected: Empty directory owned by uid 1001

---

## Task 6: Update docker-compose.yml Image Tag for API

**Files:**

- Modify: `docker-compose.yml`

**Step 1: Update API image tag to use main branch**

The docker-compose.yml currently references `:latest` but dev uses `:main` tags:

```yaml
rackula-api:
  profiles: [persist]
  image: ghcr.io/rackulalives/rackula-api:main
```

**Step 2: Commit**

```bash
git add docker-compose.yml
git commit -m "fix: use :main tag for API image in dev"
```

---

## Task 7: Push and Verify Deployment

**Files:**

- None

**Step 1: Push all changes**

```bash
git push origin main
```

**Step 2: Watch deployment**

```bash
gh run list --workflow=deploy-dev.yml --limit 1 --watch
```

Expected: All jobs pass (build-and-deploy, build-api, deploy, smoke-test)

**Step 3: Verify persistence works**

```bash
curl -sf https://d.racku.la/api/layouts
```

Expected: `[]` (empty array, not error)

---

## Verification Checklist

- [ ] API sidecar image exists at `ghcr.io/rackulalives/rackula-api:main`
- [ ] Frontend shows "New Layout" / "Open Layout" start screen
- [ ] Creating a layout persists it to the API
- [ ] Refreshing the browser shows the persisted layout
- [ ] `curl https://d.racku.la/api/layouts` returns layout list
- [ ] Health check passes: `curl https://d.racku.la/health` returns OK
