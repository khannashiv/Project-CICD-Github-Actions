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

    #### How the Kubernetes Deployment Update Works

        The "Update Kubernetes Deployment" stage:
            1. Runs only on pushes to the main branch
            2. Uses a shell script to update the image reference in the Kubernetes deployment file
            3. Commits and pushes the updated deployment file back to the repository
            4. This ensures that the Kubernetes manifest always references the latest image
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

Reference Docs:
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Variables Reference](https://docs.github.com/en/actions/reference/variables-reference)
- [GitHub Actions Metadata Syntax](https://docs.github.com/en/actions/reference/metadata-syntax-for-github-actions)
- [Storing Information in Variables](https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables) 
- [About the Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#about-the-container-registry)
- [Working with the Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Listing All Images in GitHub Container Registry](https://stackoverflow.com/questions/73879886/how-to-list-all-images-from-an-account-in-github-container-registry)
- [Trivy Action](https://github.com/aquasecurity/trivy-action)

---

## Outcomes of Github actions cicd pipelines.

- ![Github-Actions-1](../../Images/Github-Actions-1.png)
- ![Github-Actions-2](../../Images/Github-Actions-2.png)
- ![Github-Actions-3](../../Images/Github-Actions-3.png)
- ![Github-Actions-4](../../Images/Github-Actions-4.png)
- ![Github-Actions-5](../../Images/Github-Actions-5.png)
- ![Github-Actions-6](../../Images/Github-Actions-6.png)
- ![Github-Actions-7](../../Images/Github-Actions-7.png)
- ![Github-Actions-8](../../Images/Github-Actions-8.png)
- ![Github-Actions-9](../../Images/Github-Actions-9.png)
- ![GHCR-1](../../Images/GHCR-1.png)
- ![GHCR-2](../../Images/GHCR-2.png)
- ![GHCR-3](../../Images/GHCR-3.png)
- ![ArgoCD-1](../../Images/ArgoCD-1.png)
- ![ArgoCD-2](../../Images/ArgoCD-2.png)
- ![ArgoCD-3](../../Images/ArgoCD-3.png)
- ![ArgoCD-4](../../Images/ArgoCD-4.png)
- ![Outcome-1](../../Images/Outcome-1.png)
- ![Outcome-2](../../Images/Outcome-2.png)
- ![Outcome-3](../../Images/Outcome-3.png)
- ![Outcome-4](../../Images/Outcome-4.png)
- ![Outcome-5](../../Images/Outcome-5.png)
- ![Outcome-6](../../Images/Outcome-6.png)
- ![Outcome-7](../../Images/Outcome-7.png)
- ![Outcome-8](../../Images/Outcome-8.png)
- ![Outcome-9](../../Images/Outcome-9.png)
- ![Outcome-10](../../Images/Outcome-10.png)

---

