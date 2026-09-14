# CI workflows

## Publish a GHCR image

`publish-ghcr.yml` builds one image, publishes it to GHCR, maintains GitHub
Actions and registry build caches, and notifies via Webhook after a
successful upload.

The caller must grant package write access and pass both Helper Tool secrets:

```yaml
permissions:
  contents: read
  packages: write

jobs:
  publish:
    uses: ITZ-Rhein-Maas-GmbH/ci-workflows/.github/workflows/publish-ghcr.yml@main
    with:
      context: .
      dockerfile: ./.docker/prod/Dockerfile
      image_suffix: app
    secrets:
      helper_webhook_url: ${{ secrets.HELPER_WEBHOOK_URL }}
      helper_webhook_secret: ${{ secrets.HELPER_WEBHOOK_SECRET }}
```

The workflow publishes branch, commit SHA and default-branch `latest` tags. A
non-draft pull request from the same repository also publishes its sanitized
head-branch tag. Draft pull requests build without publishing.
