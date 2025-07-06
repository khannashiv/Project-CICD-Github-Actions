# GitHub Actions Variables Cheat Sheet

## 1. Default/Built-in Variables (GitHub Provided)
Automatically available in all workflows:

| Variable            | Example             | Usage in Workflow                                 |
|---------------------|---------------------|---------------------------------------------------|
| `github.repository` | `owner/repo`        | `IMAGE_NAME: ${{ github.repository }}`            |
| `github.actor`      | `khannashiv`        | Docker login username                             |
| `github.ref`        | `refs/heads/main`   | `if: github.ref == 'refs/heads/main'`             |
| `github.workflow`   | `CI/CD Pipeline`    | Used for notifications                            |
| `github.sha`        | `a1b2c3d`           | Used in Docker tags (`type=sha`)                  |

**Full List:** [GitHub Docs - Default Environment Variables](https://docs.github.com/en/actions/learn-github-actions/variables#default-environment-variables)

---

## 2. User-Defined Variables

### A. Environment Variables (Job/Workflow Level)
Defined in `env:` sections:

```yaml
env:
    REGISTRY: ghcr.io  # Hardcoded value
    IMAGE_NAME: ${{ github.repository }}  # Derived from context

jobs:
    docker:
        env:
            TRIVY_SEVERITY: "CRITICAL,HIGH"  # Job-specific
```

### B. Step Outputs
Capture output from one step, use in another:

```yaml
steps:
    - id: meta
        uses: docker/metadata-action@v5
        # Outputs: tags, labels, version

    - name: Use output
        run: echo ${{ steps.meta.outputs.tags }}
```

---

## 3. Secrets
Secure variables stored in repo/organization settings:

```yaml
steps:
    - uses: docker/login-action@v3
        with:
            password: ${{ secrets.TOKEN }}  # From repo secrets
```
> **Best Practice:** Never hardcode secrets!

---

## 4. Job Outputs
Pass values between jobs:

```yaml
jobs:
    docker:
        outputs:
            image_tag: ${{ steps.set_output.outputs.image_tag }}
    
    update-k8s:
        needs: [docker]
        steps:
            - run: echo ${{ needs.docker.outputs.image_tag }}
```

---

## 5. Contextual Variables
Dynamic workflow information:

```yaml
steps:
    - run: echo "Event was triggered by ${{ github.event_name }}"
    # Possible values: push, pull_request, schedule, etc.
```

---

## Variable Precedence

When the same name exists in multiple places (highest to lowest):

1. **Job-level env**
2. **Workflow-level env**
3. **Repository/organization secrets**
4. **Default variables**

---

## Key Examples from Your Workflow

### 1. Docker Tag Generation

```yaml
tags: |
    type=sha,format=long,prefix=sha-
```
Uses implicit `github.sha` through metadata-action.

### 2. Kubernetes Update

```yaml
NEW_IMAGE="${REGISTRY}/${GITHUB_REPOSITORY}:${IMAGE_TAG}"
```
Combines:
- User-defined `REGISTRY` (`env`)
- Default `GITHUB_REPOSITORY`
- Job output `IMAGE_TAG`

### 3. Conditional Execution

```yaml
if: github.ref == 'refs/heads/main'
```
Uses default `github.ref`.

---

## When to Use Each Type

| Use Case                   | Recommended Variable Type   |
|----------------------------|----------------------------|
| Sensitive data             | Secrets                    |
| Reusable constants         | Workflow/job env           |
| Step-to-step communication | Step outputs               |
| Job-to-job communication   | Job outputs                |
| GitHub platform info       | Default variables          |

---

## Pro Tips

**Debug variables:**
```yaml
- name: Print all contexts
    run: echo '${{ toJSON(github) }}'
```

**Complex expressions:**
```yaml
env:
    DEPLOY_ENV: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
```

**Organization-level variables:**  
Set in org settings for cross-repo consistency.

### Reference Docs

- [GitHub Actions Reference](https://docs.github.com/en/actions/reference)
- [Variables Reference](https://docs.github.com/en/actions/reference/variables-reference)
- [Metadata Syntax for GitHub Actions](https://docs.github.com/en/actions/reference/metadata-syntax-for-github-actions)
- [Storing Information in Variables](https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables)
- [GitHub Actions Contexts and Expressions](https://docs.github.com/en/actions/learn-github-actions/contexts#about-contexts-and-expressions)
- [GitHub Actions Environment Variables](https://docs.github.com/en/actions/learn-github-actions/contexts#about-contexts-and-expressions)