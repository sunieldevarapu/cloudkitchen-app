# CloudKitchen GitHub Actions

CI/CD for the CloudKitchen platform. The pipeline **builds, scans, and pushes**
container images, then **updates image tags in git** for ArgoCD to deploy. It
never runs `helm upgrade` or `kubectl apply` — deployment is ArgoCD's job.

## Workflows

| File            | Trigger                              | Purpose                                                                                   |
| --------------- | ------------------------------------ | ----------------------------------------------------------------------------------------- |
| `ci.yaml`       | push to `main`, PRs (on service dirs) | Matrix build of all 9 services → Trivy image scan → push to ECR → bump dev tags (main only) |
| `trivy-fs.yaml` | pull requests                        | Trivy filesystem + config/IaC scan; uploads SARIF; fails PR on HIGH/CRITICAL               |

### `ci.yaml` jobs

1. **build** — matrix over the 9 services (`auth-service`, `user-service`,
   `restaurant-service`, `menu-service`, `order-service`, `payment-service`,
   `delivery-service`, `notification-service`, `frontend`). Builds run in
   parallel. Each: build image locally → Trivy scan (fail on HIGH/CRITICAL) →
   push `:<short-sha>` and `:latest` to ECR. **PRs build + scan only; they do
   not push.**
2. **update-gitops** (`needs: build`, push to `main` only) — uses `yq` to set the
   full `image:` string for each service in `helm/cloudkitchen/values.yaml`, then
   commits and pushes the change. ArgoCD reconciles the new tags into the cluster.

## Required configuration

Configure these under **Settings → Secrets and variables → Actions**.

### Secrets

| Secret                  | Required | Description                                                                                                                         |
| ----------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | Yes      | Access key ID of the IAM user used by CI. The user needs ECR push permissions (see policy below).                                   |
| `AWS_SECRET_ACCESS_KEY` | Yes      | Secret access key paired with `AWS_ACCESS_KEY_ID`.                                                                                  |
| `GITOPS_TOKEN`          | Optional | PAT / fine-grained token with `contents: write` on this repo, used to push the GitOps tag commit. Only needed if branch protection blocks the default `GITHUB_TOKEN`. Falls back to `GITHUB_TOKEN`. |

### Variables

| Variable       | Required | Example                                                  | Description                                              |
| -------------- | -------- | -------------------------------------------------------- | -------------------------------------------------------- |
| `ECR_REGISTRY` | Yes      | `123456789012.dkr.ecr.us-east-1.amazonaws.com`           | ECR registry host = `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com`. Images push to `<ECR_REGISTRY>/cloudkitchen/<service>`. |

`AWS_REGION` (`us-east-1`) and the ECR image prefix (`cloudkitchen`) are hard-coded
as `env` in `ci.yaml`.

## AWS IAM user prerequisites (one-time)

1. Create a dedicated IAM user for CI (e.g. `cloudkitchen-github-ci`) with
   **programmatic access** (no console login).
2. Attach a policy granting ECR push/pull:
   `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`,
   `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`,
   `ecr:PutImage`, `ecr:BatchGetImage`.
3. Create an access key for that user and store the two values as the GitHub
   Secrets `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
4. Ensure each `cloudkitchen/<service>` ECR repository exists (create them via
   the Terraform `ecr` module in `terraform/`).

> Security note: static long-lived keys are convenient but should be rotated
> regularly and scoped to ECR only. OIDC (federated, keyless) is the more secure
> alternative if you switch later.

## values.yaml image convention

`update-gitops` maps each service directory (`auth-service`) to its **camelCase**
key (`authService`) and sets the full `image:` string in
`helm/cloudkitchen/values.yaml`:

```yaml
authService:
  image: <ECR_REGISTRY>/cloudkitchen/auth-service:<short-sha>
userService:
  image: <ECR_REGISTRY>/cloudkitchen/user-service:<short-sha>
# ... one block per service (+ frontend)
```

This matches the flat, explicit `values.yaml` used by `helm/cloudkitchen`.

## Notes

- The GitOps commit message contains `[skip ci]` to avoid an infinite CI loop
  (the tag-bump commit only touches `helm/cloudkitchen/values.yaml`, which is
  outside the `paths` filter, but `[skip ci]` is a belt-and-braces safeguard).
- Pinned action versions (`aquasecurity/trivy-action@0.24.0`, `mikefarah/yq@v4.44.3`,
  etc.) should be bumped deliberately.
