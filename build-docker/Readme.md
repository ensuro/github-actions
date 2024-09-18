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

There are other optional arguments for controlling testing and tags:

- `additional_tag`: Set an additional tag, for instance from a manual run input
- `run_tests`: Set tu true to run the tests (requires `test_script`)
- `test_script`: A script to run to test the built image. Similar usage to `run: ` in a standard job
- `test_build_args`: Additional build args for the image built for testing. Example `test_build_args: APP_ENV=ci`

## Version number

The version number will be exposed as a build arg named `DOCKER_METADATA_OUTPUT_VERSION`. You can use this for setting the version on your package. For example, in a `Dockerfile` for a python app that uses `setuptool_scm` you can do:

```Dockerfile
ARG DOCKER_METADATA_OUTPUT_VERSION
ENV SETUPTOOLS_SCM_PRETEND_VERSION=$DOCKER_METADATA_OUTPUT_VERSION
```

### To create a service account and obtain its credentials

```bash
ACCOUNT_NAME=gh-ens-simulator
REPOSITORY=ensuro
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
