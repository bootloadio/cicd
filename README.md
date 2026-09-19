# bootload CI/CD

GitHub Actions and example pipelines for deploying to
[bootload](https://bootload.io), the container host that runs every app in its
own Firecracker microVM.

Two actions live here:

- `setup-bootload` installs the CLI on the runner and, given an API key,
  authenticates it for every later step.
- `deploy` rolls a service onto an image and waits until it is healthy,
  rolling back if it is not.

## Without an API key

A deploy hook is a signed URL that can roll one service onto another tag of
the image that service already runs. It cannot deploy a different image, reach
another service, or read anything. That makes it safe to keep in a CI secret
in a way a full API key is not.

Create it once:

```
bootload deploy-hook create web --name "github actions"
```

Store the URL and secret it prints, then:

```yaml
- uses: bootloadio/cicd/deploy@v1
  with:
    hook-url: ${{ secrets.BOOTLOAD_DEPLOY_HOOK }}
    hook-secret: ${{ secrets.BOOTLOAD_DEPLOY_SECRET }}
    tag: ${{ github.sha }}
```

The action signs each call with `X-Bootload-Signature: t=<unix>,v1=<hmac>`, so
the secret never travels in a URL where a proxy log could keep it.

The trade is that a hook holds no read scope, so the job cannot wait for the
rollout to go healthy. Use an API key when you want that.

## With an API key

```yaml
- uses: bootloadio/cicd/setup-bootload@v1
  with:
    token: ${{ secrets.BOOTLOAD_TOKEN }}
    project: production

- uses: bootloadio/cicd/deploy@v1
  with:
    service: web
    project: production
    image: registry.bootload.io/acme/production/web:${{ github.sha }}
```

Mint the key narrowly:

```
bootload token create ci \
  --scope projects:read --scope services:read --scope services:write \
  --scope deployments:read --scope deployments:write \
  --scope registry:push --scope registry:pull \
  --project production
```

`projects:read` is what lets `--project production` resolve by name. A key
restricted with `--project` can only touch that project, including its
registry namespace.

## Examples

Complete pipelines you can copy:

- [`examples/github/deploy-hook.yml`](examples/github/deploy-hook.yml) builds,
  pushes, and rolls with a deploy hook.
- [`examples/github/api-key.yml`](examples/github/api-key.yml) does the same
  with an API key, and waits for health with rollback.
- [`examples/github/pull-request-check.yml`](examples/github/pull-request-check.yml)
  builds a pull request and reports what a deploy would change, without
  deploying.
- [`examples/gitlab/.gitlab-ci.yml`](examples/gitlab/.gitlab-ci.yml) is the
  GitLab version, signing the hook call with `openssl`.

## setup-bootload inputs

| Input | Meaning |
|---|---|
| `token` | API key, exported as `BOOTLOAD_TOKEN` for later steps |
| `project` | project name or id to select as the default |
| `org` | organization slug or id to select as the default |
| `api` | API base URL; leave empty for production |
| `expect-version` | fail if the installed CLI is not this version |

`expect-version` is an assertion, not a download pin. bootload serves one
current CLI release, so pinning is not possible. Asserting means your pipeline
breaks loudly when the CLI changes under it, instead of changing behaviour
quietly.

## deploy inputs

| Input | Meaning |
|---|---|
| `hook-url`, `hook-secret` | deploy-hook mode; no API key needed |
| `service` | service name or id (API-key mode) |
| `image` | full image reference to deploy |
| `tag` | tag only; the repository is the one the service already runs |
| `project`, `token`, `api` | as above |
| `wait`, `timeout` | wait for health, API-key mode only |
| `rollback-on-failure` | roll back when the new version never gets healthy |

Send no `image` and no `tag` to re-pull what is already configured, which is
what a mutable tag like `:latest` wants.

## Notes

bootload runs x86-64 (linux/amd64) images. Build with
`docker build --platform linux/amd64` on an arm64 machine.

Push images to `registry.bootload.io`. Pulls then happen inside the datacenter
instead of over the internet, and they are never rate-limited the way
anonymous Docker Hub pulls are.

Full documentation: [CI/CD pipelines](https://bootload.io/docs/ci-cd-pipelines/).

## Contributing

These actions are generated from the `actions/` directory of the bootload
platform repository, so they cannot drift from the CLI they drive. File issues
here; fixes land upstream and are published back to this repo.

MIT licensed.
