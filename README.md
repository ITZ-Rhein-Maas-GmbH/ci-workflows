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

The build receives `APP_VERSION` containing the caller's Git commit SHA. A
Dockerfile can declare `ARG APP_VERSION` and store it in an image environment
variable for application release tracking.

Set the optional `target` input to publish a named Dockerfile stage. Use a
distinct `image_suffix` for each image so their tags and build caches stay separate.
Omitting `target` builds the final stage.

### Build caches

Each image keeps its own GitHub Actions cache scope. Branch builds also export
all intermediate stages to GHCR with `mode=max`. The default branch uses
`:buildcache`; other branches use `:buildcache-branch-<first 16 SHA-256 hex characters of the full Git ref>`.
Hashing the full ref avoids collisions between names such as `design/jero` and
`design-jero`. Each branch imports its own registry cache and the default branch's
cache. Keep these tags when applying package cleanup rules.

The registry cache survives GitHub Actions' seven-day idle cache eviction.
Different branches and images write separate cache tags, so parallel builds
do not replace each other's cache manifests. Pull requests and tag builds do
not write registry caches; fork pull requests do not access private registry
caches. GitHub Actions' own branch access restrictions still apply to GHA caches.

For images with shared Dockerfile stages, set `cache_from_image_suffixes` to
additional suffixes, one per line. For example, an `app` build can import `cli`
and a `cli` build can import `app`. This only adds cache reads; each build still
writes its own cache. Missing caches on the first build are expected. Concurrent
cold builds may both compile shared stages before either exports its cache.

Changing a base image digest or a build instruction invalidates the dependent
layers even when the cache is available. Floating base tags continue to pick up
upstream updates; persistent caching does not prevent these rebuilds.

### Private submodules

Submodule checkout is disabled by default. Callers can enable recursive
checkout and either pass a token directly or let the workflow create a
short-lived GitHub App token.

For a GitHub App, install the app on every repository involved in the checkout
and grant it read-only repository contents access:

```yaml
jobs:
  publish:
    uses: ITZ-Rhein-Maas-GmbH/ci-workflows/.github/workflows/publish-ghcr.yml@main
    with:
      checkout_submodules: true
      checkout_app_id: ${{ vars.CHECKOUT_APP_ID }}
      checkout_app_repositories: |
        application-repository
        private-submodule-repository
    secrets:
      checkout_app_private_key: ${{ secrets.CHECKOUT_APP_PRIVATE_KEY }}
      helper_webhook_url: ${{ secrets.HELPER_WEBHOOK_URL }}
      helper_webhook_secret: ${{ secrets.HELPER_WEBHOOK_SECRET }}
```

#### Set up the GitHub App

Do not store an installation access token as an Actions secret. Installation
tokens expire after one hour. This workflow creates a token for each job and
revokes it when the job finishes.

1. [Register a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)
   under the organization that owns the caller and submodule repositories.
2. Give the app `Read-only` access under **Repository permissions > Contents**.
   Disable webhooks and restrict installation to the app owner's account unless
   the app needs to be installed elsewhere.
3. [Install the app](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)
   on the caller repository and every private submodule repository. Prefer
   **Only select repositories**.
4. Copy the numeric App ID from the app settings page.
5. Under **Private keys**, select **Generate a private key** and save the
   downloaded PEM file securely. GitHub documents key generation and rotation
   in [Managing private keys for GitHub Apps](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps).
6. Add `CHECKOUT_APP_ID` as an Actions variable and the full PEM file as the
   `CHECKOUT_APP_PRIVATE_KEY` Actions secret in the caller repository:

   ```bash
   gh variable set CHECKOUT_APP_ID \
     --repo OWNER/CALLER_REPOSITORY \
     --body 'NUMERIC_APP_ID'

   gh secret set CHECKOUT_APP_PRIVATE_KEY \
     --repo OWNER/CALLER_REPOSITORY \
     < /path/to/github-app-private-key.pem
   ```

Organization-level variables and secrets also work when their repository
access includes the caller. The caller passes the App ID and private key to the
reusable workflow explicitly. The workflow uses
[`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
to generate the short-lived installation token.

Publish changes to this shared workflow before publishing a caller that uses
new inputs or secrets. For callers that reference `@main`, merge the
`ci-workflows` change first.

Alternatively, pass a PAT or another valid credential as the optional
`checkout_token` secret. When no checkout credential is supplied, the workflow
uses the caller repository's `GITHUB_TOKEN`.
