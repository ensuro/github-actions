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

You will need a secret with json credentials for a google service account. Instructions for creating this secret are in the next section.

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

jobs:
  build-push:
    runs-on: ubuntu-latest

    steps:
      - name: Build and push
        uses: ensuro/github-actions/build-docker@main
        with:
          image: "myrepo/myimage"
          google_credentials: "${{ secrets.GOOGLE_CREDENTIALS }}"
```

### To create a service account and obtain its credentials

```bash
ACCOUNT_NAME=gh-ens-simulator
PROJECT_ID=solid-range-319205

# Create the account
gcloud --project "${PROJECT_ID}" iam service-accounts create ${ACCOUNT_NAME} \
    --description="Github actions ${ACCOUNT_NAME}" \
    --display-name="Github actions ${ACCOUNT_NAME}"

# Grant permissions on pkg.dev
for role in "roles/artifactregistry.admin" \
             "roles/artifactregistry.repoAdmin" \
             "roles/iam.serviceAccountTokenCreator" \
             "roles/serviceusage.apiKeysViewer"
do
    gcloud projects add-iam-policy-binding ${PROJECT_ID} --member \
        serviceAccount:"${ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
        --role "${role}" \
        --no-user-output-enabled --quiet
done


# Export account key (BE CAREFUL NOT TO COMMIT THIS FILE!)
gcloud iam service-accounts keys create "credentials-${ACCOUNT_NAME}.json" \
    --iam-account=${ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com

```

Credentials will be in a `credentials-*.json` file. **Be careful not to commit the file to any repo**.

You have to paste the contents on that file in a secret named `GOOGLE_CREDENTIALS` in your repo under Settings -> Secrets -> Actions
