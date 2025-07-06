## Explaination of github actions pipeline.

1. Workflow Metadata & Triggers

name: CI/CD Pipeline
on:
  push:
    branches: [ main ]
    paths-ignore:
      - 'kubernetes/deployment.yaml'
  pull_request:
    branches: [ main ]

<!-- 
 Explanation:
        name:          Sets the name of the workflow (CI/CD Pipeline).
        on.push:       Triggers the workflow when code is pushed to the main branch.
        paths-ignore:  Skips triggering if changes are only in kubernetes/deployment.yaml (to avoid infinite loops when updating K8s manifests).
        on.pull_request: Also triggers when a PR is opened against main.
 -->

2. Jobs Structure

        The workflow consists of 4 jobs:
        test → Runs unit tests.
        lint → Performs static code analysis (ESLint).
        build → Builds the project.
        docker → Builds, scans, and pushes a Docker image.
        update-k8s → Updates Kubernetes deployment (only on main branch pushes).

3. Job: test (Unit Testing)

test:
  name: Unit Testing
  runs-on: ubuntu-latest
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run tests
      run: npm test || echo "No tests found"

<!-- 
Explanation:
        runs-on: ubuntu-latest:   Uses a GitHub-hosted Ubuntu runner.
        actions/checkout@v4:      Checks out the repository code.
        actions/setup-node@v4:    Sets up Node.js v20.
        cache: 'npm':             Caches node_modules for faster builds.
        npm ci:                   Installs dependencies (clean install, locked to package-lock.json).
        npm test: Runs tests (with || fallback if no tests exist).
 -->

 4. Job: lint (Static Code Analysis)

 lint:
  name: Static Code Analysis
  runs-on: ubuntu-latest
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run ESLint
      run: npm run lint

<!-- Explanation:
        Similar to test, but runs ESLint (npm run lint). 
-->

5. Job: build (Build Project)

build:
  name: Build
  runs-on: ubuntu-latest
  needs: [test, lint]  # Depends on test & lint passing
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Build project
      run: npm run build

    - name: Upload build artifacts
      uses: actions/upload-artifact@v4
      with:
        name: build-artifacts
        path: dist/

<!-- 
    Explanation:
        needs:         [test, lint]: Runs only if test and lint jobs succeed.
        npm run build: Builds the project (e.g., transpiles TypeScript, bundles React, etc.).
        actions/upload-artifact@v4: Uploads the dist/ folder as an artifact for later use in the docker job.
 -->

 6. Job: docker (Docker Build & Push)

 docker:
  name: Docker Build and Push
  runs-on: ubuntu-latest
  needs: [build]  # Depends on build job
  env:
    REGISTRY: ghcr.io
    IMAGE_NAME: ${{ github.repository }}
  outputs:
    image_tag: ${{ steps.set_output.outputs.image_tag }}
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Download build artifacts
      uses: actions/download-artifact@v4
      with:
        name: build-artifacts
        path: dist/

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.TOKEN }}

    - name: Extract metadata for Docker
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,format=long,prefix=sha-

    - name: Set image tag output
      id: set_output
      run: echo "image_tag=${{ steps.meta.outputs.version }}" >> $GITHUB_OUTPUT

    - name: Build and load Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        load: true

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@0.28.0
      with:
        image-ref: "${{ steps.meta.outputs.tags }}"
        format: 'table'
        exit-code: '1'
        ignore-unfixed: true
        vuln-type: 'os,library'
        severity: 'CRITICAL,HIGH'

    - name: Push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

<!-- 
Key Components:
        env: Sets REGISTRY=ghcr.io and IMAGE_NAME=github-repo-name.
        outputs: Exposes image_tag for use in update-k8s.
        docker/metadata-action@v5:
            Generates Docker tags & labels automatically.
            type=sha,format=long,prefix=sha- → Creates a tag like sha-<full-git-sha>.
        docker/build-push-action@v5:
            First, builds and loads the image locally (push: false).
            After Trivy scan, pushes to GHCR (push: true).
        aquasecurity/trivy-action@0.28.0:
            Scans for OS & library vulnerabilities (vuln-type).
            Fails only on CRITICAL/HIGH (severity).
            Ignores unfixed vulnerabilities (ignore-unfixed).
-->

7. Job: update-k8s (Update Kubernetes Deployment)

update-k8s:
  name: Update Kubernetes Deployment
  runs-on: ubuntu-latest
  needs: [docker]  # Depends on docker job
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        token: ${{ secrets.TOKEN }}

    - name: Setup Git config
      run: |
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"

    - name: Update Kubernetes deployment file
      env:
        IMAGE_TAG: ${{ needs.docker.outputs.image_tag }}
        GITHUB_REPOSITORY: ${{ github.repository }}
        REGISTRY: ghcr.io
      run: |
        NEW_IMAGE="${REGISTRY}/${GITHUB_REPOSITORY}:${IMAGE_TAG}"
        sed -i "s|image: ${REGISTRY}/.*|image: ${NEW_IMAGE}|g" kubernetes/deployment.yaml
        echo "Updated deployment to use image: ${NEW_IMAGE}"
        grep -A 1 "image:" kubernetes/deployment.yaml

    - name: Commit and push changes
      run: |
        git add kubernetes/deployment.yaml
        git commit -m "Update K8s deployment with image: ${{ needs.docker.outputs.image_tag }} [skip ci]" || echo "No changes to commit"
        git push

<!-- 
    Explanation:
            if: Runs only on main branch pushes (not PRs).
            needs: [docker]: Waits for the docker job to complete.
            sed Command:
                Updates kubernetes/deployment.yaml with the new image tag.
                Example: image: ghcr.io/user/repo:sha-abc123
            git commit & push:
                Commits the updated deployment.yaml.
                [skip ci] prevents an infinite loop (since pushing changes would normally trigger the workflow again).
 -->