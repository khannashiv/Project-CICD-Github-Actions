## GitHub Actions Pipeline Explanation

### 1. Workflow Overview

This pipeline automates:

- ✅ **Testing:** Unit tests  
- ✅ **Linting:** Code quality checks  
- ✅ **Building:** Compiles the application  
- ✅ **Dockerization:** Builds, scans, and pushes container images  
- ✅ **Kubernetes Deployment:** Updates K8s manifests  
- ✅ **Cleanup:** Maintains only the last 2 workflow runs and Docker images  

---

### 2. Trigger Rules

```yaml
on:
    push:
        branches: [main]
        paths-ignore:
            - 'kubernetes/deployment.yaml'  # Prevents loops when K8s manifest updates
            - '**/*.md'                    # Skips CI for Markdown file changes
    pull_request:
        branches: [main]
```

> **Key Behavior:**  
> - Runs on pushes to `main` except for:
>   - Changes to `deployment.yaml` (prevents infinite loops)
>   - Changes to any `.md` files (docs-only changes)
> - Runs on all PRs to `main` (regardless of file changes)

---

### 3. Job Dependencies

```mermaid
graph TD
        A[test] --> C[build]
        B[lint] --> C
        C --> D[docker]
        D --> E[update-k8s]
        E --> F[cleanup]
```

- **Parallel:** `test` + `lint`
- **Serial:** `build` → `docker` → `update-k8s` → `cleanup`

---

### 4. Key Job Explanations

#### A. Docker Job

- **Purpose:** Build, scan, and push Docker images to GHCR
- **Critical Steps:**
    1. **Metadata Generation:** Creates tags like `sha-<commit-hash>`
    2. **Trivy Scanning:** Blocks push if critical/high vulnerabilities found
    3. **Image Cleanup:** Keeps only last 2 tagged images & removes all untagged

<details>
<summary>Cleanup Logic</summary>

```bash
# Keep last 2 tagged images
TAGGED_DIGESTS=$(... | tail -n +3)

# Delete ALL untagged images
UNTAGGED_DIGESTS=$(...)
```
</details>

---

#### B. Kubernetes Update Job

- **Conditions:** Only runs on push to `main`
- **What it does:**
    1. Updates `deployment.yaml` with new image tag
    2. Commits with `[skip ci]` to prevent workflow loops

---

#### C. Cleanup Job

- **Purpose:** Prevent workflow run clutter
- **Behavior:**  
    - Keeps last 2 completed runs (any status)  
    - Deletes older runs via GitHub API

---

### 5. Smart Optimization Features

| Feature              | Implementation                                 | Benefit                              |
|----------------------|------------------------------------------------|--------------------------------------|
| Docs Ignore          | `paths-ignore: '**/*.md'`                      | Skips CI for documentation changes   |
| K8s Loop Prevention  | `paths-ignore: deployment.yaml` + `[skip ci]`  | Avoids infinite CI loops             |
| Docker Storage       | Automated cleanup of old images                 | Prevents GHCR storage bloat          |
| Workflow History     | Keep last 2 runs                                | Clean GitHub Actions UI              |

---

### 6. Security Controls

- **Image Scanning:**  
    - Trivy checks for OS/library vulnerabilities  
    - Fails on critical/high severity issues

- **Secret Handling:**  
    - Uses `secrets.TOKEN` for registry auth  
    - Minimal permissions needed

---

### 7. Failure Recovery

- `if: always()` in cleanup steps ensures:
    - Image cleanup runs even if earlier steps fail
    - Workflow history gets pruned regardless of job status


## How the Kubernetes Deployment Update Works

The "Update Kubernetes Deployment" stage:

1. Runs only on pushes to the main branch
2. Uses a shell script to update the image reference in the Kubernetes deployment file
3. Commits and pushes the updated deployment file back to the repository
4. This ensures that the Kubernetes manifest always references the latest image

## Required Secrets

The workflow requires the following GitHub secrets:

- `GITHUB_TOKEN` - Automatically provided by GitHub Actions, used for pushing to the repository and the container registry

## Continuous Deployment

For full continuous deployment, you would need to:

1. Set up a Kubernetes operator like Flux or ArgoCD to watch for changes in the repository
2. Configure it to automatically apply changes to the Kubernetes manifests
3. This would complete the CI/CD pipeline by automatically deploying the new image to your Kubernetes cluster

## Manual Deployment

If you're not using a GitOps approach with an operator, you can manually apply the updated deployment:

```bash
kubectl apply -f kubernetes/deployment.yaml
```

Or set up a webhook to trigger the deployment when the manifest is updated.