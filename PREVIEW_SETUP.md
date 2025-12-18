# Chart Preview Setup Guide

Welcome to Chart Preview! This guide will help you configure automated chart preview deployments for your repository.

## 🚀 Quick Start

Your Chart Preview system is almost ready! Follow these steps to complete the setup:

### 1. Create GitHub Workflow (Copy & Paste)

⚠️ **Manual Step Required**: Due to GitHub App security restrictions, the workflow file could not be automatically placed in `.github/workflows/`.

**Please follow these steps in order:**

**Step A: Merge this PR first**
1. Review and merge this PR to get the setup guide into your repository

**Step B: Add the workflow file directly to main**
2. Go to your repository on GitHub
3. Click **Add file** → **Create new file**
4. Name the file: `.github/workflows/chart-preview.yml`
5. Paste the following content:

```yaml
name: Chart Preview
on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths-ignore:
      - '.github/**'
      - '**.md'

permissions:
  contents: read
  packages: write
  statuses: write

jobs:
  # Job 1: Build and push Docker image
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ghcr.io/${{ github.repository }}
      digest: ${{ steps.push.outputs.digest }}
      tags: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=pr
            type=sha,prefix=pr-${{ github.event.pull_request.number }}-
            type=raw,value=pr-${{ github.event.pull_request.number }}

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          provenance: false
          sbom: false
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Update commit status (image-build)
        if: success()
        run: |
          curl -X POST \
               -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
               -H "Accept: application/vnd.github+json" \
               -H "X-GitHub-Api-Version: 2022-11-28" \
               "https://api.github.com/repos/${{ github.repository }}/statuses/${{ github.sha }}" \
               -d '{
                 "state": "success",
                 "context": "image-build",
                 "description": "Docker image built and pushed successfully"
               }'

      - name: Update commit status on failure (image-build)
        if: failure()
        run: |
          curl -X POST \
               -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
               -H "Accept: application/vnd.github+json" \
               -H "X-GitHub-Api-Version: 2022-11-28" \
               "https://api.github.com/repos/${{ github.repository }}/statuses/${{ github.sha }}" \
               -d '{
                 "state": "failure",
                 "context": "image-build",
                 "description": "Docker image build failed"
               }'

  # Job 2: Deploy preview environment
  deploy-preview:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy Preview Environment
        run: |
          cat > payload.json <<EOF
          {
            "repo": "${{ github.repository }}",
            "pr": ${{ github.event.pull_request.number }},
            "chart": {
              "source": "oci://ghcr.io/${{ github.repository_owner }}/helm-chart",
              "version": "1.0.0"
            },
            "values": {
              "image": {
                "repository": "${{ needs.build.outputs.image }}",
                "digest": "${{ needs.build.outputs.digest }}"
              },
              "env": {
                "APP_VERSION": "PR-${{ github.event.pull_request.number }}"
              }
            },
            "head": {
              "owner": "${{ github.repository_owner }}",
              "repo": "${{ github.event.repository.name }}",
              "sha": "${{ github.event.pull_request.head.sha }}"
            }
          }
          EOF

          # Note: TTL is not specified - will use tier-appropriate defaults:
          # Free: 2 hours, Pro: 12 hours, Team: 24 hours, Enterprise: configurable
          # You can override by adding "ttl": HOURS to the payload above

          # Note: The chart source above works out of the box for most use cases.
          # For easier management across multiple workflows, you can optionally:
          # 1. Configure "Default Chart Source" in your repository settings dashboard
          # 2. Remove the "chart" section above - dashboard settings will be used automatically

          echo "Deploying preview environment..."
          curl -fsSL -X POST \
            -H "Authorization: Bearer ${{ secrets.PREVIEW_TOKEN }}" \
            -H "Content-Type: application/json" \
            -H "User-Agent: GitHub-Actions-Preview-Deploy" \
            --data @payload.json \
            "https://api.previews.chart-preview.com/previews" || {
              echo "Preview deployment failed. Check your configuration:"
              echo "1. Verify your PREVIEW_TOKEN secret is set"
              echo "2. Check your Helm chart configuration"
              echo "3. Ensure your chart source URL is correct"
              echo "4. Review the setup guide in PREVIEW_SETUP.md"
              exit 1
            }```

6. Commit directly to `main` branch (not via PR)

⚠️ **Important**: Create the workflow file directly on `main`, not via a pull request. If you open a PR to add the workflow, it will trigger itself and fail.

### 2. Configure Your Helm Chart

The workflow includes a default chart source that works out of the box:

```yaml
"chart": {
  "source": "oci://ghcr.io/chart-preview/helm-chart",
  "version": "1.0.0"
}
```

**To customize your chart source**, you have several options:

**Option A: Edit the Workflow File** (Works Immediately)

Update the chart source in `.github/workflows/chart-preview.yml` to point to your chart:
- **OCI Registry**: `oci://ghcr.io/chart-preview/my-app-chart`
- **Helm Repository**: `bitnami/nginx` (requires repo to be pre-configured on platform)
- **TGZ URL**: `https://example.com/charts/myapp-1.0.0.tgz`

**Option A2: Git Chart Flow** (Pro/Team only - No packaging required!)

Deploy directly from a Helm chart in your Git repository without packaging to OCI/TGZ:

```json
{
  "repo": "chart-preview/grafana-helmcharts",
  "pr": ${{ github.event.pull_request.number }},
  "git": {
    "owner": "chart-preview",
    "repo": "grafana-helmcharts",
    "ref": "${{ github.event.pull_request.head.sha }}",
    "chartPath": "helm/"  // Path to Chart.yaml in your repo
  },
  "values": { ... },
  "head": { ... }
}
```

**Benefits of Git Chart Flow:**
- No need to package and publish charts to a registry
- Chart changes in your PR are automatically deployed
- Works with private repositories (uses GitHub App authentication)

**Option B: Centralize in Dashboard** (Easier Management)

For easier management across multiple workflows:
1. Go to your Chart Preview dashboard: `https://previews.chart-preview.com/dashboard`
2. Navigate to **Repositories** → Click **Configure** on your repository
3. Set **Default Chart Source** to your chart location
4. Remove the `"chart"` section from your workflow file
5. The workflow will automatically use the dashboard setting

✅ **Benefit of Dashboard Config**: Update chart source once, applies to all workflows.

**Note**: If both are configured, the workflow file takes precedence.

### 3. Set Up Your Docker Image Build

The workflow builds and pushes images to GitHub Container Registry. Ensure your repository has:

- [ ] A `Dockerfile` in the root directory
- [ ] Proper image build configuration

**For organization repositories**, enable GHCR write access:
1. Go to your organization Settings → Actions → General
2. Under **Workflow permissions**, select **Read and write permissions**
3. Click Save

Without this, the workflow will fail with `installation not allowed to Create organization package`.

### 4. Configure Chart Preview Token

Your Chart Preview token has been generated and is ready to use:
- **Token**: `rpt_UASgBYvB.UASgBYvB3rfaThI2HHCCYxqKxAARJmsSgfSTQj0SHto`
- **Security**: Add this as a repository secret named `PREVIEW_TOKEN`

⚠️ **Important**: Keep this token secure! It provides access to your Chart Preview deployments.

**To add the secret:**
1. Go to your repository Settings
2. Navigate to Secrets and variables → Actions
3. Click "New repository secret"
4. Name: `PREVIEW_TOKEN`
5. Value: The token above

### 5. Customize Your Values

Update the values section in the workflow to match your chart:

```yaml
"values": {
  "image": {
    "repository": "${{ needs.build.outputs.image }}",
    "digest": "${{ needs.build.outputs.digest }}"
  },
  # Add your custom values here
  "database": {
    "enabled": true
  },
  "ingress": {
    "enabled": true
  }
}
```

### 6. Private Container Images (Optional)

If your Docker images are stored in a private registry (like private GHCR packages), you need to:

1. **Add imagePullSecrets to your Helm chart** - Your chart's deployment template should support imagePullSecrets
2. **Pass the secrets in your workflow** - Add the `secrets` field to create the pull secret

**Step A: Update your values to include imagePullSecrets:**

```yaml
"values": {
  "imagePullSecrets": [{"name": "ghcr-secret"}],
  "image": {
    "repository": "${{ needs.build.outputs.image }}",
    "digest": "${{ needs.build.outputs.digest }}"
  }
}
```

**Step B: Add the secrets field to create the Kubernetes secret:**

```yaml
"secrets": {
  "ghcr-secret": "{\"type\":\"kubernetes.io/dockerconfigjson\",\"data\":{\".dockerconfigjson\":\"${{ secrets.GHCR_DOCKERCONFIG }}\"}}"
}
```

**Step C: Create the GHCR_DOCKERCONFIG secret:**

Generate the base64-encoded Docker config:

```bash
# Step 1: Base64 encode username:token
AUTH=$(echo -n "USERNAME:GITHUB_PAT" | base64)

# Step 2: Create Docker config JSON and base64 encode the whole thing
echo -n "{\"auths\":{\"ghcr.io\":{\"auth\":\"$AUTH\"}}}" | base64
```

Add the output as a repository secret named `GHCR_DOCKERCONFIG`.

**Note:** Your Helm chart's deployment template must include imagePullSecrets support:

```yaml
spec:
  {{- with .Values.imagePullSecrets }}
  imagePullSecrets:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  containers:
    ...
```

## 📊 Monitoring & Debugging

### Check Preview Status

View your preview environments:
- Dashboard: `https://previews.chart-preview.com/dashboard`
- API: `https://api.previews.chart-preview.com/previews?repo=grafana-helmcharts`

### Common Issues

**Build Failures:**
- Check Docker build logs in Actions
- Verify Dockerfile syntax
- Ensure all dependencies are available

**Deployment Failures:**
- Verify Helm chart syntax: `helm lint your-chart/`
- Check resource requirements
- Review tier limits and quotas

## 🎯 Next Steps

1. [ ] Merge this PR to get the setup guide into your repository
2. [ ] Add the workflow file to `.github/workflows/chart-preview.yml` on `main`
3. [ ] Add the `PREVIEW_TOKEN` secret to your repository
4. [ ] Create a test PR to verify the setup

---

*Generated automatically by the Chart Preview Platform*
