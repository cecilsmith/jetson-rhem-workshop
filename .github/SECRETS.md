# Repository secrets

`.github/workflows/build-jetson-image.yml` builds the Jetson bootc image on a
GitHub-hosted **ARM64** runner. Everything it needs to authenticate comes from
repository secrets — nothing credential-bearing lives in the repo.

`.github/workflows/secret-scan.yml` is the other half: it scans the full git
history (gitleaks, `.gitleaks.toml`) and the tracked tree for credentials that
slipped in, and fails if a file like `config.yaml` or `pull-secret` ever becomes
tracked.

## Required secrets

| Secret                     | What it is                                                                                                                          | Where to get it                                                             |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `REDHAT_REGISTRY_USERNAME` | Registry service account username for `registry.redhat.io`. Looks like `12345678\|myserviceaccount`.                                | <https://access.redhat.com/terms-based-registry/> → **New service account** |
| `REDHAT_REGISTRY_PASSWORD` | That service account's token.                                                                                                       | Same page → **Token** tab                                                   |
| `RH_USERNAME`              | Red Hat Customer Portal login that `subscription-manager register` uses inside the build.                                           | Your Red Hat account                                                        |
| `RH_PASSWD`                | That account's password.                                                                                                            | Your Red Hat account                                                        |
| `DEVICE_USERNAME`          | Login account created on the device, e.g. `redhat`. Not strictly secret; kept here so the image contents aren't pinned in the repo. | You choose                                                                  |
| `DEVICE_USER_PASSWD`       | Password for that device account. **This one is baked into the shipped image** — treat the released tarball accordingly.            | You choose                                                                  |

## Optional secrets
<!-- 
| Secret            | When you need it                                                                                                                                                                                                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OCP_PULL_SECRET` | Only for the MicroShift Containerfiles (`ContainerFile-Jetson-1.0.2-base`), which pre-pull OpenShift release images. Download from <https://console.redhat.com/openshift/install/pull-secret> and paste the whole JSON blob. Not needed by `ContainerFile-Jetson-1.3.0-docker-compose`. | --> |

`GITHUB_TOKEN` is provided automatically — you do not create it. It is used to
push the layer cache to `ghcr.io` and to create the release.

## Use a registry service account, not your portal password

`REDHAT_REGISTRY_*` should be a **registry service account**, not your Red Hat
login. Service accounts are revocable individually and are scoped to registry
pulls, so a leak does not hand over your Red Hat account.

## Setting them up

### Web UI

1. Go to `https://github.com/cecilsmith/jetson-rhem-workshop/settings/secrets/actions`
   (repo → **Settings** → **Secrets and variables** → **Actions**).
2. **New repository secret**, one per row in the tables above.
3. Name must match exactly — the `preflight` job lists any that are missing and
   stops before the ARM runner is allocated.

### `gh` CLI

```bash
gh secret set REDHAT_REGISTRY_USERNAME --repo cecilsmith/jetson-rhem-workshop
gh secret set REDHAT_REGISTRY_PASSWORD --repo cecilsmith/jetson-rhem-workshop
gh secret set RH_USERNAME              --repo cecilsmith/jetson-rhem-workshop
gh secret set RH_PASSWD                --repo cecilsmith/jetson-rhem-workshop
gh secret set DEVICE_USERNAME          --repo cecilsmith/jetson-rhem-workshop
gh secret set DEVICE_USER_PASSWD       --repo cecilsmith/jetson-rhem-workshop
```

Each prompts for the value on stdin, so it never lands in your shell history.
For the multi-line pull secret, read it from the file instead:

```bash
gh secret set OCP_PULL_SECRET --repo cecilsmith/jetson-rhem-workshop < ./pull-secret
```

Verify (names and update times only — values are never readable back):

```bash
gh secret list --repo cecilsmith/jetson-rhem-workshop
```

## Also turn on GitHub's built-in scanning

Free for public repositories, and it catches things before they are ever pushed:

`Settings` → `Advanced Security` → enable **Secret scanning** and **Push
protection**.

## How the build consumes them

Credentials are written to `$RUNNER_TEMP` and passed with `podman build
--secret`, mounted at `/run/secrets/<id>` inside the build. They are **never**
passed with `--build-arg`: buildah records build-arg values in the image
history, which would publish them inside the released tarball. A post-build step
greps the image history and config for each secret value and fails the job if
one appears.

The build also runs with `--squash`, so the entitlement certificates that the
final `subscription-manager clean` deletes are actually gone from the archive
rather than merely whiteouted in a lower layer.

## Operational notes

- **Orphaned registrations.** Each cache-miss build calls `subscription-manager
  register --auto-attach`, consuming an entitlement, and returns it via
  `unregister` in the last layer. A build that fails in between leaks a
  registration. Check <https://access.redhat.com/management/systems> now and
  then and delete stale `localhost`-ish entries.
- **Secrets and forks.** Secrets are not exposed to `pull_request` runs from
  forks, which is why the build workflow triggers only on `push` to `main` and
  `workflow_dispatch`.
- **Rotation.** After rotating `RH_PASSWD`, update the secret and re-run. The
  registration layer is cached for 168h, so an already-cached build will not
  re-authenticate until the cache expires or the Containerfile changes.
