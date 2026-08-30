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
* Run: `sudo ./1_configure-firewall-and-registry.sh`
* Run: `sudo ./2_rhem-server-build-commands.sh`

Once these scripts are complete, you should be able to open the Flight Control web UI at `https://LAPTOP_HOSTNAME`.  The login credentials are admin/admin unless you changed them in the `2_rhem-server-build-commands.sh` script.

# Post Installation Steps

With Flight Control now installed, you can generate the `config.yaml` file that you'll need to place on your managed devices to allow them to enroll to the server:

* `flightctl login https://LAPTOP_HOSTNAME:3443 --insecure-skip-tls-verify --web`
* `flightctl certificate request --signer=enrollment --expiration=365d --output=embedded > config.yaml`

Add the following line to the end of the `config.yaml` file and save it for use later!

* `spec-fetch-interval: 5s`

# Managed Device Image Creation

The bootc container image is now built by GitHub Actions on a **free, native ARM64
runner**, so you no longer need to stand up your own ARM build server just to
produce the image. See [`.github/SECRETS.md`](.github/SECRETS.md) for the
repository secrets the workflow needs.

> **You still need an aarch64 machine for the ISO step.** CI produces the
> *container image*; converting it into an installable ISO uses
> `bootc-image-builder`, which has to run on ARM hardware. A Jetson or any
> aarch64 box work.

## Option 1: Download a prebuilt image (recommended)

Each successful build publishes a [release](../../releases) tagged with the
build date:

* Image name: `localhost/jetson-flightctl-<FLIGHTCTL_VERSION>-<VARIANT>:<YYYY-MM-DD>`
* Release asset: `jetson-flightctl-<FLIGHTCTL_VERSION>-<VARIANT>-<YYYY-MM-DD>.tar.zst`

Download that `.tar.zst` plus `SHA256SUMS`, then:

```bash
sha256sum -c SHA256SUMS
zstd -d jetson-flightctl-1.3.0-docker-compose-YYYY-MM-DD.tar.zst
sudo podman load -i jetson-flightctl-1.3.0-docker-compose-YYYY-MM-DD.tar
```

GitHub attaches *Source code (zip)* and *Source code (tar.gz)* to every release
automatically and they cannot be removed — those are the repository, not the
image. Ignore them.

If the image exceeds 2 GiB compressed it is split into `.part00`, `.part01`, …
Reassemble with `cat ...part* > image.tar.zst` before decompressing; the release
notes spell this out per build.

## Option 2: Build it yourself

See [`managed-device-artifacts/jetson-bootc-build-commands`](managed-device-artifacts/jetson-bootc-build-commands)
for the full command reference. In short, on an aarch64 host:

```bash
mkdir -p .secrets && chmod 700 .secrets
printf %s 'YOUR_REDHAT_LOGIN'    > .secrets/rh_username
printf %s 'YOUR_REDHAT_PASSWORD' > .secrets/rh_password

sudo podman build \
  --secret id=rh_username,src=.secrets/rh_username \
  --secret id=rh_password,src=.secrets/rh_password \
  --build-arg USERNAME=redhat --build-arg USER_PASSWD=redhat \
  --squash \
  -t localhost/jetson-flightctl-1.3.0-docker-compose:$(date -u +%Y-%m-%d) \
  -f ./ContainerFile-Jetson-1.3.0-docker-compose .
```

Red Hat credentials are passed as **build secrets, never as `--build-arg`**:
`podman` records build-arg values in the image history, so a build arg would ship
your Red Hat password inside the image. `--squash` matters for the same reason —
without it the entitlement certificates that `subscription-manager clean` deletes
in the final layer remain readable in the layer beneath.

## Device login

The account baked into the image is **public on purpose** so anyone flashing a
device can log in:

| Username | Password |
| -------- | -------- |
| `redhat` | `redhat` |

It is in the `wheel` group, so it can `sudo`. Override with the `USERNAME` /
`USER_PASSWD` build args locally, or the `DEVICE_USERNAME` / `DEVICE_USER_PASSWD`
repository variables in CI. **Change these before deploying a device anywhere
outside a lab.**

## Which Containerfile gets built

The `ContainerFile-*` symlink at the repository root selects the active build.
It currently points at the 1.3.0 docker-compose variant. Repoint it to change
what CI builds — the image name is derived from the filename, so
`ContainerFile-Jetson-1.3.0-docker-compose` produces
`jetson-flightctl-1.3.0-docker-compose`.

## Repository layout

* `ContainerFile-Jetson-1.3.0-docker-compose` (root symlink): selects the active build target.
* `managed-device-artifacts/ContainerFile-Jetson-1.3.0-docker-compose`: the current image. Includes the Flight Control agent and the `docker-compose` plugin binary — no Docker daemon is installed.
* `managed-device-artifacts/jetson-bootc-build-commands`: command reference for building the image and turning it into an ISO.
* `managed-device-artifacts/rhem.ks`: sample kickstart for the `mkksiso` step.
* `managed-device-artifacts/legacy/`: superseded Containerfiles, kept for reference only and not built by CI.

## Continuous integration

* `.github/workflows/build-jetson-image.yml` — builds on `ubuntu-24.04-arm`, verifies no credentials leaked into the image, saves it, and publishes the release. Runs on pushes that touch a Containerfile, or on demand via **Actions → Build Jetson bootc image → Run workflow**.
* `.github/workflows/secret-scan.yml` — scans the full git history with gitleaks and fails if credential material or files like `config.yaml` ever become tracked.

> **Version skew:** `2_rhem-server-build-commands.sh` installs Flight Control
> **1.0.2**, while the current device image ships agent **1.3.0**. Keep the
> server and agent versions aligned, or expect enrollment trouble.
