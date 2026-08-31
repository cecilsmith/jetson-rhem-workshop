# Intention

This repository is intended to hold artifacts for creating a Flight Control (AKA Red Hat Edge Manager) 
homelab.  If you want to learn more about this project, check out its repo [here](https://github.com/flightctl/flightctl/blob/main/docs/user/introduction.md).
Also included in this repository are commands and ContainerFiles for creating NVIDIA Jetson Red Hat Enterprise Linux Image Mode (bootc) images 
that will be used to flash edge devices managed by Flight Control.

# Prerequisites
 
* A laptop running Red Hat Enterprise Linux 9.x or 10.x
* DHCP server on your home network.  This can be installed on the laptop if needed per [these instructions](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_networking_infrastructure_services/index#setting-up-the-dhcp-service-for-subnets-directly-connected-to-the-dhcp-server_providing-dhcp-services) or you can just let your home network router assign IP addresses.
* Ideally, use wired networking.  Wireless networking will involve additional steps outside the scope of this repo.
* One or more NVIDIA Orin-based devices to manage (Nano, NX, AGX)

# Initial Laptop Setup

Start with a server+GUI RHEL 9.x or 10.x installation, with the "container management" software group selected.  During RHEL
installation, configure a regular user with `sudo` privileges on the host.  

> **Make sure to set a hostname on the system other than localhost via:**
> 
> `sudo hostnamectl set-hostname <LAPTOP_HOSTNAME>`

I have successfully deployed this setup on a NUC with 12 GB RAM and an Intel
N97 processor, which are pretty minimal specifications, but you can likely go
even lower on RAM if needed.

Once the system is back up:

* `sudo dnf install -y git`
* `git clone https://github.com/cecilsmith/jetson-rhem-workshop`

# Flight Control Server Installation

* Go into the `./jetson-rhem-workshop` directory that you cloned
* Run: `sudo ./server/1_configure-firewall-and-registry.sh`
* Run: `sudo ./server/2_rhem-server-build-commands.sh`

Once these scripts are complete, you should be able to open the Flight Control web UI at `https://LAPTOP_HOSTNAME`.  The login credentials are admin/admin unless you changed them in the `server/2_rhem-server-build-commands.sh` script.

# Post Installation Steps

With Flight Control now installed, you can generate the `config.yaml` file that you'll need to place on your managed devices to allow them to enroll to the server:

* `flightctl login https://LAPTOP_HOSTNAME:3443 --insecure-skip-tls-verify --web`
* `flightctl certificate request --signer=enrollment --expiration=365d --output=embedded > config.yaml`

Add the following line to the end of the `config.yaml` file and save it for use later!

* `spec-fetch-interval: 5s`

# Managed Device Image Creation

Both images are built by GitHub Actions on a **free, native ARM64 runner**, so
you do not need your own ARM build server to produce them. See
[`.github/SECRETS.md`](.github/SECRETS.md) for the repository secrets the
workflow needs, and [`BUILDING.md`](BUILDING.md) for the full command reference.

> **You still need an aarch64 machine for the ISO step.** CI produces the
> *container images*; converting one into an installable ISO uses
> `bootc-image-builder`, which has to run on ARM hardware. A Jetson or any
> aarch64 box works.

## Two images per release

Every release ships two variants, identical except for the compose
implementation. Neither installs a Docker daemon.

| Image                   | Compose implementation                             |
| ----------------------- | -------------------------------------------------- |
| `jetson-docker-compose` | Docker Compose v2 plugin (`docker-compose-plugin`) |
| `jetson-podman-compose` | `podman-compose` from EPEL                         |

Names are stable; the Flight Control version and build date live in the **tag**:

```
localhost/jetson-docker-compose:1.3.0-2026-08-30
                                ^^^^^ ^^^^^^^^^^
                                agent  build date
```

Keeping the version out of the name matters for fielded devices: `bootc upgrade`
pulls the reference the device was switched to, so a name that changed on every
version bump would break the upgrade path.

Releases are tagged `v<FLIGHTCTL_VERSION>-<YYYY-MM-DD>`, e.g. `v1.3.0-2026-08-30`.

## When builds happen

* **A new Flight Control release.** A daily scheduled job asks
  `rpm.flightctl.io` for the newest GA `flightctl-agent` and builds only when
  that version has not been released here yet. The RPM repository is the trigger
  rather than the flightctl GitHub releases page because the RPM is what
  `dnf install` actually consumes — a GitHub release can land before the RPM is
  published.
* **A change to `images/Containerfile`** pushed to `main`.
* **On demand** via **Actions → Build Jetson images → Run workflow**, where you
  can pin a specific version or force a rebuild.

## Download a prebuilt image

Grab the `.tar.zst` for the variant you want plus its `.sha256`:

```bash
VARIANT=jetson-docker-compose      # or jetson-podman-compose
sha256sum -c ${VARIANT}-1.3.0-2026-08-30.sha256
zstd -d ${VARIANT}-1.3.0-2026-08-30.tar.zst
sudo podman load -i ${VARIANT}-1.3.0-2026-08-30.tar
```

GitHub attaches *Source code (zip)* and *Source code (tar.gz)* to every release
automatically and they cannot be removed — those are the repository, not the
images. Ignore them.

## Device login

The account baked into the images is **public on purpose** so anyone flashing a
device can log in. Refer to the specific release for login information. The CI runner uses the repo's variables for this, but by default they are:

| Username | Password |
| -------- | -------- |
| `redhat` | `redhat` |

It is in the `wheel` group, so it can `sudo`. Override with the `USERNAME` /
`USER_PASSWD` build args locally, or the `DEVICE_USERNAME` / `DEVICE_USER_PASSWD`
repository variables in CI. **Change these before deploying a device anywhere
outside a lab.**

## Repository layout

```
server/     Flight Control server setup for the laptop/NUC
images/     Containerfile (living), rhem.ks, archive/ (frozen snapshots)
BUILDING.md Command reference: build an image, turn it into an ISO
```

`images/Containerfile` is the single, living definition of both variants — they
differ only by the `COMPOSE_PROVIDER` build argument. It floats to current
versions; CI pins `FLIGHTCTL_VERSION` per build so each release matches its tag.

`images/archive/*.Containerfile` are frozen, fully self-contained snapshots of
past images, kept for reproducibility and never edited.

## Pinned by the supplier

RHEL **9.4** and JetPack **6.1** are hardcoded because that is the only
combination NVIDIA publishes a Jetson RPM repository for — `rhel-9.5`,
`rhel-9.6`, `rhel-10*` and `jp7.x` all 404. The Flight Control RPM repository is
EPEL 9 only. Revisit if NVIDIA publishes a RHEL 10 path.

## Continuous integration

* `.github/workflows/build-images.yml` — builds both variants on
  `ubuntu-24.04-arm`, verifies each image (no leaked credentials, no entitlement
  certificates, `--squash` took effect, correct agent version, correct compose
  provider), and publishes one release carrying both.
* `.github/workflows/secret-scan.yml` — scans the full git history with gitleaks
  and fails if credential material or files like `config.yaml` become tracked.

