# Repository secrets

`.github/workflows/build-jetson-image.yml` builds the Jetson bootc image on a
GitHub-hosted **ARM64** runner. Everything it needs to authenticate comes from
repository secrets — nothing credential-bearing lives in the repo.

`.github/workflows/secret-scan.yml` is the other half: it scans the full git
history (gitleaks, `.gitleaks.toml`) and the tracked tree for credentials that
slipped in, and fails if a file like `config.yaml` ever becomes tracked.

## Required secrets

| Secret                     | What it is                                                                                                                          | Where to get it                                                             |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `REDHAT_REGISTRY_USERNAME` | Registry service account username for `registry.redhat.io`. Looks like `12345678\|myserviceaccount`.                                | <https://access.redhat.com/terms-based-registry/> → **New service account** |
| `REDHAT_REGISTRY_PASSWORD` | That service account's token.                                                                                                       | Same page → **Token** tab                                                   |
| `RH_USERNAME`              | Red Hat Customer Portal login that `subscription-manager register` uses inside the build.                                           | Your Red Hat account                                                        |
| `RH_PASSWD`                | That account's password.                                                                                                            | Your Red Hat account                                                        |

## The device login is a variable, not a secret

The account people log in with after flashing a device is **published on
purpose** — a workshop attendee with a freshly flashed Jetson needs to be able to
get in. It lives in repository *variables*, which are visible in the Actions UI
and printed in both the job summary and the release notes:

| Variable | Default | What it is |
|---|---|---|
| `DEVICE_USERNAME` | `redhat` | Login account created on the device, in the `wheel` group |
| `DEVICE_USER_PASSWD` | `redhat` | That account's password |

Both are optional. Leave them unset and the build uses the defaults above, which
match the `ARG` defaults in the Containerfile.

```bash
gh variable set DEVICE_USERNAME    --body redhat --repo cecilsmith/jetson-rhem-workshop
gh variable set DEVICE_USER_PASSWD --body redhat --repo cecilsmith/jetson-rhem-workshop
```

These are passed as ordinary `--build-arg` values, which is correct here for the
same reason it is wrong for the Red Hat credentials: build args land in the image
history, and here that history *is* the documentation. Anyone who can reach a
flashed device can log into it, so change these before deploying a device
anywhere outside a lab.

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
```

Each prompts for the value on stdin, so it never lands in your shell history.
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
rather than merely whiteouted in a lower layer. This is enforced, not assumed:

1. The flag combination is probed against a throwaway build first, and the job
   fails in seconds if podman rejects it.
2. After the build, the layer count is compared against the base image to prove
   `--squash` took effect.
3. The release step refuses to publish unless that assertion passed and, on a
   non-private repository, refuses outright when it did not.

## Keep the build cache private

`--cache-to` exports the **intermediate** layers to
`ghcr.io/<owner>/<repo>/buildcache`, including the layer created immediately
after `subscription-manager register` — entitlement certificates intact.
`--squash` does nothing about this, because the cache is written before the
final image is committed.

The build asserts the cache is not anonymously readable and fails if it is. If
that check ever fires: set the package to private under
**your profile → Packages → buildcache → Package settings**, then re-register
the affected system so the exposed entitlement is rotated.

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
