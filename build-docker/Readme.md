# Build docker image

This action does the following tasks:

- Clone the repo
- Authenticate with Google's Artifact Registry
- Build the image
- [optional] run tests using the built image
- Upload the image to pkg.dev

The image will be tagged with as much info from the repo as possible (tag, branch, commit hash, parsed version from tag, etc.). Check the outputs or the registry for more info.

The main point of this action is to have a standard build that can be shared accross many repositories with minimal repetition.

## Usage

OIDC authentication through Workload Identity Federation is the recommended way to authenticate. It does not require storing a long-lived service account key in GitHub.

```yaml
name: Build and Push Docker Image to Google Artifact Registry

# All these triggers are supported and can be used in combination
on:
  # Manual run from github
  workflow_dispatch:

  push:
    # Don't run on all commits on all branches (you can remove -ignore to enable that behaviour)
    branches-ignore:
      - "*"

    # Run on all tags like v0.0.1 or v1.3.2
    tags:
      - v*

# Required for OIDC authentication
permissions:
  contents: read
  id-token: write

jobs:
  build-push:
    runs-on: ubuntu-latest

    steps:
      - name: Build and push
        uses: ensuro/github-actions/build-docker@main
        with:
          # Full image: us-docker.pkg.dev/PROJECT_ID/GAR_REPOSITORY/IMAGE
          image: "my-project/ensuro/ens-offchain"
          workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
          service_account: "gh-ensuro@my-project.iam.gserviceaccount.com"
```

In the `image` input, `ensuro` is the **Google Artifact Registry (GAR) repository** and `ens-offchain` is the image name (usually the same as the GitHub repository name). With the default registry (`us-docker.pkg.dev`), this example pushes to:

```text
us-docker.pkg.dev/my-project/ensuro/ens-offchain
```

`id-token: write` is mandatory for OIDC, and it must be set in the calling workflow because a composite action cannot grant permissions by itself. See [Authentication setup for Artifact Registry](#authentication-setup-for-artifact-registry) for the one-time Google Cloud setup.

When both OIDC inputs and `google_credentials` are provided, OIDC is used.

### Deprecated JSON key authentication

The previous `google_credentials` input is still supported as a fallback, but it is deprecated because it relies on a long-lived key:

```yaml
      - name: Build and push
        uses: ensuro/github-actions/build-docker@main
        with:
          image: "my-project/ensuro/ens-offchain"
          google_credentials: "${{ secrets.GOOGLE_CREDENTIALS }}"
```

There are other optional arguments for controlling testing and tags:

- `registry`: Image registry to use (default: `us-docker.pkg.dev`)
- `build_args`: Build arguments passed to buildx
- `additional_tag`: Set an additional tag, for instance from a manual run input
- `run_tests`: Set to true to run the tests (requires `test_script`)
- `test_script`: A script to run to test the built image. Similar usage to `run: ` in a standard job
- `test_build_args`: Additional build args for the image built for testing. Example `test_build_args: APP_ENV=ci`
- `build_context`: Directory to use as build context (default: `.`)

## Version number

The version number will be exposed as a build arg named `DOCKER_METADATA_OUTPUT_VERSION`. You can use this for setting the version on your package. For example, in a `Dockerfile` for a python app that uses `setuptool_scm` you can do:

```Dockerfile
ARG DOCKER_METADATA_OUTPUT_VERSION
ENV SETUPTOOLS_SCM_PRETEND_VERSION=$DOCKER_METADATA_OUTPUT_VERSION
```

## Authentication setup for Artifact Registry

The OIDC setup is split into three scopes, so you only repeat what is necessary:

| Scope | What to configure | Repeated |
| --- | --- | --- |
| Google Cloud project | Workload Identity Pool and OIDC provider | Once per Google Cloud project |
| GAR repository | Service account and `roles/artifactregistry.writer` on the GAR repository | Once per GAR repository (for example `ensuro`) |
| GitHub repository | Permission for the GitHub repository to impersonate the GAR repository's service account | Once per GitHub repository (for example `ensuro/ens-offchain`) |

> **Note:** `REPOSITORY` in these commands always means the **Google Artifact Registry repository** (for example `ensuro`), not the GitHub repository.

Artifact Registry IAM cannot be scoped to an individual image: all images in a GAR repository share the permissions granted on that repository. If you need per-image isolation, use a separate GAR repository for each image.

### Once per Google Cloud project

This creates the Workload Identity Pool and one OIDC provider for your GitHub organization. If they already exist, skip the `create` commands and only retrieve the provider resource name.

```bash
PROJECT_ID=solid-range-319205
GITHUB_ORG=ensuro
POOL_NAME=github-ci-pool
PROVIDER_NAME=github-provider

gcloud iam workload-identity-pools create "$POOL_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --display-name="GitHub Actions Pool"

WORKLOAD_IDENTITY_POOL_ID=$(gcloud iam workload-identity-pools describe "$POOL_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --format="value(name)")

gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_NAME" \
  --display-name="GitHub ${GITHUB_ORG}" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

WORKLOAD_IDENTITY_PROVIDER=$(gcloud iam workload-identity-pools providers describe "$PROVIDER_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_NAME" \
  --format="value(name)")

echo "$WORKLOAD_IDENTITY_PROVIDER"
```

The provider resource name has this form:

```text
projects/123456789/locations/global/workloadIdentityPools/github/providers/github
```

Use it as the `workload_identity_provider` input. It is the same for every GitHub repository in the organization.

### Once per GAR repository

This creates one service account per GAR repository and grants it write access to that repository only. If the service account already exists, skip the `create` command and only add the repository binding.

```bash
PROJECT_ID=solid-range-319205
REPOSITORY=ensuro   # Google Artifact Registry repository name
LOCATION=us
ACCOUNT_NAME=gh-oicd-ensuro

gcloud iam service-accounts create "$ACCOUNT_NAME" \
  --project="$PROJECT_ID" \
  --display-name="GitHub Actions (${REPOSITORY} Artifact Registry)"

SERVICE_ACCOUNT_EMAIL="${ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud artifacts repositories add-iam-policy-binding "$REPOSITORY" \
  --project="$PROJECT_ID" \
  --location="$LOCATION" \
  --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
  --role="roles/artifactregistry.writer"
```

`roles/artifactregistry.writer` allows pushing images and using the registry-based build cache. Because the binding is on the GAR repository, the account cannot push to other GAR repositories in the project.

Use the service account email as the `service_account` input. All GitHub repositories that push images into this GAR repository can share the same service account.

If you need even tighter permissions, create a custom role instead of `roles/artifactregistry.writer`. For Docker push and registry cache it needs at least `artifactregistry.repositories.downloadArtifacts`, `artifactregistry.repositories.uploadArtifacts` and the `artifactregistry.tags.*` permissions.

### Once per GitHub repository

This is the only step needed for each new GitHub repository that pushes an image to an existing GAR repository. It allows that specific GitHub repository to impersonate the GAR repository's service account.

```bash
PROJECT_ID=solid-range-319205
GITHUB_ORG=ensuro
GITHUB_REPO=ens-offchain
ACCOUNT_NAME=gh-oicd-ensuro
POOL_NAME=github-ci-pool

WORKLOAD_IDENTITY_POOL_ID=$(gcloud iam workload-identity-pools describe "$POOL_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --format="value(name)")

SERVICE_ACCOUNT_EMAIL="${ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts add-iam-policy-binding "$SERVICE_ACCOUNT_EMAIL" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"
```

For example, for `https://github.com/ensuro/ens-offchain`, run the command with `GITHUB_REPO=ens-offchain`. This grants only that GitHub repository permission to use the service account configured above, which can push images such as:

```text
us-docker.pkg.dev/my-project/ensuro/ens-offchain
```

Repeat this step with a different `GITHUB_REPO` for every GitHub repository that should push to the `ensuro` GAR repository. No JSON key file is created.

### Legacy: JSON key credentials (deprecated)

```bash
ACCOUNT_NAME=gh-ens-simulator
REPOSITORY=ensuro   # Google Artifact Registry repository name
LOCATION=us

# Create the account
SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts create $ACCOUNT_NAME \
    --display-name="Github actions ${ACCOUNT_NAME}" \
    --format="value(email)")

# Grant permissions on the artifacts repo
gcloud artifacts repositories add-iam-policy-binding $REPOSITORY \
   --location=$LOCATION \
   --member=serviceAccount:"$SERVICE_ACCOUNT_EMAIL" \
   --role=roles/artifactregistry.writer

# Grant the account permissions to create access tokens for itself
gcloud iam service-accounts add-iam-policy-binding "$SERVICE_ACCOUNT_EMAIL"\
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role=roles/iam.serviceAccountTokenCreator

# Export account key (BE CAREFUL NOT TO COMMIT THIS FILE!)
gcloud iam service-accounts keys create "credentials-${ACCOUNT_NAME}.json" \
    --iam-account="$SERVICE_ACCOUNT_EMAIL"

```

Credentials will be in a `credentials-*.json` file. **Be careful not to commit the file to any repo**.

You have to paste the contents on that file in a secret named `GOOGLE_CREDENTIALS` in your repo under Settings -> Secrets -> Actions

It's highly recommended that you convert the json to a single line to paste it into the secret:

```sh
jq -c < credentials-$ACCOUNT_NAME.json
```
