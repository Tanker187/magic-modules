# Cloud Build Setup Guide

This document describes how Cloud Build is configured for the `magic-modules` repository and how to manage it.

## Overview

Cloud Build is integrated with this repository to:
- Validate Terraform configurations on every pull request
- Run automated tests before merging
- Deploy infrastructure changes to GCP projects
- Maintain audit logs of all infrastructure changes

## Architecture

```
GitHub PR/Push
    ↓
GitHub Actions Workflow
    ↓
Cloud Build Trigger (gtm-kqqwvx2-zgi2z)
    ↓
Terraform Plan/Validate
    ↓
Test Suite
    ↓
Terraform Apply (main branch only)
```

## Configuration Files

### `cloudbuild.yaml`
Located at repository root. Defines the build steps:

1. **tf-validate**: Validates Terraform syntax
2. **tf-fmt**: Checks code formatting
3. **tf-plan**: Creates execution plan
4. **run-tests**: Executes test suite
5. **tf-apply**: Applies changes (main branch only)

### `.github/workflows/cloud-build-trigger.yml`
GitHub Actions workflow that:
- Triggers on PR creation/update to `main` or `develop`
- Triggers on push to `main`
- Authenticates to GCP using Workload Identity Federation
- Submits build to Cloud Build
- Streams build logs back to GitHub

## Setup Instructions

### 1. Create GCP Service Account

```bash
# Create service account
gcloud iam service-accounts create magic-modules-ci \
  --display-name="Magic Modules CI/CD" \
  --project=gtm-kqqwvx2-zgi2z

# Grant required roles
gcloud projects add-iam-policy-binding gtm-kqqwvx2-zgi2z \
  --member=serviceAccount:magic-modules-ci@gtm-kqqwvx2-zgi2z.iam.gserviceaccount.com \
  --role=roles/cloudbuild.builds.editor

gcloud projects add-iam-policy-binding gtm-kqqwvx2-zgi2z \
  --member=serviceAccount:magic-modules-ci@gtm-kqqwvx2-zgi2z.iam.gserviceaccount.com \
  --role=roles/compute.admin
```

### 2. Configure Workload Identity Federation

```bash
# Create Workload Identity Pool
gcloud iam workload-identity-pools create github \
  --project=gtm-kqqwvx2-zgi2z \
  --location=global \
  --display-name="GitHub"

# Create Workload Identity Provider
gcloud iam workload-identity-pools providers create-oidc github \
  --project=gtm-kqqwvx2-zgi2z \
  --location=global \
  --workload-identity-pool=github \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
  --issuer-uri=https://token.actions.githubusercontent.com

# Get the provider resource name
export WORKLOAD_IDENTITY_PROVIDER=$(gcloud iam workload-identity-pools providers describe github \
  --project=gtm-kqqwvx2-zgi2z \
  --location=global \
  --workload-identity-pool=github \
  --format='value(name)')
```

### 3. Create Service Account IAM Binding

```bash
gcloud iam service-accounts add-iam-policy-binding \
  magic-modules-ci@gtm-kqqwvx2-zgi2z.iam.gserviceaccount.com \
  --project=gtm-kqqwvx2-zgi2z \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/gtm-kqqwvx2-zgi2z/locations/global/workloadIdentityPools/github/attribute.repository/Tanker187/magic-modules"
```

### 4. Configure GitHub Secrets

Add the following secrets to your GitHub repository (`Settings > Secrets and variables > Actions`):

```
GCP_PROJECT_ID: gtm-kqqwvx2-zgi2z
GCP_SERVICE_ACCOUNT: magic-modules-ci@gtm-kqqwvx2-zgi2z.iam.gserviceaccount.com
GCP_WORKLOAD_IDENTITY_PROVIDER: <WORKLOAD_IDENTITY_PROVIDER from step 2>
```

### 5. Create Cloud Build Trigger

```bash
gcloud builds create-github-trigger magic-modules \
  --repo-name=magic-modules \
  --repo-owner=Tanker187 \
  --branch-pattern=main \
  --build-config=cloudbuild.yaml \
  --project=gtm-kqqwvx2-zgi2z
```

## Workflow

### For Pull Requests
1. Developer creates PR with Terraform changes
2. GitHub Actions triggers Cloud Build
3. Cloud Build validates and plans changes
4. Results posted to PR as a check
5. PR cannot be merged until checks pass (with branch protection)

### For Main Branch
1. PR is merged to `main`
2. GitHub Actions triggers Cloud Build
3. Cloud Build validates, plans, and applies changes
4. Terraform state is updated in GCS
5. Build logs are available in Cloud Build console

## Monitoring and Troubleshooting

### View Build Logs
```bash
# List recent builds
gcloud builds list --project=gtm-kqqwvx2-zgi2z --limit=10

# View specific build
gcloud builds log <BUILD_ID> --project=gtm-kqqwvx2-zgi2z --stream
```

### Common Issues

**Build fails with permission denied:**
- Verify service account has required roles
- Check Workload Identity Pool configuration
- Ensure GitHub secrets are set correctly

**Terraform state conflicts:**
- Check for concurrent builds
- Verify state lock configuration
- Review recent terraform applies

**Build timeout:**
- Increase timeout in `cloudbuild.yaml`
- Optimize Terraform configuration
- Check for external API delays

## Security Best Practices

1. **Limit permissions**: Use least-privilege IAM roles
2. **Encrypt state**: Enable encryption for Terraform state in GCS
3. **Audit logging**: Enable Cloud Build audit logs
4. **Code review**: Require approvals before merging to `main`
5. **Secrets management**: Use Google Secret Manager for sensitive data

## Permissions

The following GitHub users can manage Cloud Build:
- `@Tanker187` (owner)
- `@googleapis/python-core-client-libraries` (upstream team)

See `.github/CODEOWNERS` for details.

## Support

For issues or questions:
1. Check Cloud Build logs: `gcloud builds log`
2. Review GitHub Actions logs
3. Check Terraform state: `terraform show`
4. Contact: @Tanker187
