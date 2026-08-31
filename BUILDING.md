# Building the managed device image

A command reference, not a script to run top to bottom.

Two ways to get a container image:

1. **Download one CI already built** — jump to [Load a released image](#load-a-released-image).
2. **Build it yourself** on an aarch64 host — [Build the image](#build-the-image).

Either way, the ISO step must run on **aarch64 hardware**: `bootc-image-builder`
runs the image it is packaging.

## Why RHEL 9.4 and JetPack 6.1 are fixed

NVIDIA publishes exactly one Jetson RPM repository for RHEL:

```
https://repo.download.nvidia.com/jetson/rhel-9.4/jp6.1/    ← the only path that exists
```

`rhel-9.5`, `rhel-9.6`, `rhel-10*` and `jp7.x` all return 404 for RHEL. JetPack
7.x exists, but only for Ubuntu/L4T. The Flight Control RPM repository is
likewise EPEL 9 only (`rpm.flightctl.io/epel/9/$basearch/`). So the base image
and driver stack are pinned by the supplier, not by choice. Revisit when NVIDIA
publishes a RHEL 10 path.

## Prerequisites

```bash
# mkksiso comes from lorax
dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm lorax

# Needed to pull the RHEL bootc base image and the image builder
podman login registry.redhat.io
```

If you are pushing to the local registry created by
`server/1_configure-firewall-and-registry.sh`, declare it insecure and restart
podman. Replace `$HOSTIP` with the address of the machine running the registry:

```bash
cat > /etc/containers/registries.conf.d/999-local-registry.conf <<EOF
[[registry]]
location = "$HOSTIP:5000"
insecure = true
EOF
```

## Load a released image

Each release carries both variants. Pick one:

| Variant | Compose implementation |
|---|---|
| `jetson-docker-compose` | Docker Compose v2 plugin (no Docker daemon) |
| `jetson-podman-compose` | `podman-compose` from EPEL |

```bash
export VARIANT=jetson-docker-compose
export VERSION=1.3.0
export DATE_TAG=2026-08-30

sha256sum -c ${VARIANT}-${VERSION}-${DATE_TAG}.sha256

# If the asset was split at the 2 GiB release limit, join the parts first:
#   cat ${VARIANT}-${VERSION}-${DATE_TAG}.tar.zst.part* > ${VARIANT}-${VERSION}-${DATE_TAG}.tar.zst

zstd -d ${VARIANT}-${VERSION}-${DATE_TAG}.tar.zst
podman load -i ${VARIANT}-${VERSION}-${DATE_TAG}.tar
```

Loads as `localhost/${VARIANT}:${VERSION}-${DATE_TAG}`. Skip to
[Build the ISO](#build-the-iso).

Ignore *Source code (zip)* and *Source code (tar.gz)* on the release page —
GitHub attaches those to every release automatically and they cannot be removed.

## Build the image

Red Hat credentials are passed as build **secrets**, never as `--build-arg`:
podman records build-arg values in the image history, so a build arg would ship
your Red Hat password inside the image.

```bash
mkdir -p .secrets && chmod 700 .secrets
printf %s 'YOUR_REDHAT_LOGIN'    > .secrets/rh_username
printf %s 'YOUR_REDHAT_PASSWORD' > .secrets/rh_password
chmod 600 .secrets/*
```

The device login is public on purpose so anyone flashing the image can log in.
Change it before deploying a device anywhere outside a lab.

```bash
export COMPOSE_PROVIDER=docker-compose        # or podman-compose
export FLIGHTCTL_VERSION=1.3.0                # blank installs whatever is newest
export DATE_TAG=$(date -u +%Y-%m-%d)
export IMAGE_NAME=localhost/jetson-${COMPOSE_PROVIDER}:${FLIGHTCTL_VERSION}-${DATE_TAG}

podman build \
  --secret id=rh_username,src=.secrets/rh_username \
  --secret id=rh_password,src=.secrets/rh_password \
  --build-arg COMPOSE_PROVIDER="${COMPOSE_PROVIDER}" \
  --build-arg FLIGHTCTL_VERSION="${FLIGHTCTL_VERSION}" \
  --build-arg USERNAME=redhat \
  --build-arg USER_PASSWD=redhat \
  --squash \
  -t "${IMAGE_NAME}" \
  -f images/Containerfile \
  .
```

`--squash` is not optional if you intend to share the result. Without it, the
entitlement certificates that the final `subscription-manager clean` deletes are
only whiteouted — the real files stay readable in the layer underneath and ship
with the image.

### Check what you built

The same four checks CI runs:

```bash
# 1. No entitlement material left behind (each directory separately: `ls -A a b`
#    prints a header per operand and looks non-empty even when both are empty)
podman run --rm --entrypoint /bin/sh "${IMAGE_NAME}" -c \
  'for d in /etc/pki/entitlement /etc/pki/consumer; do ls -A "$d"; done'

# 2. --squash took effect: exactly one more layer than the base image
podman image inspect "${IMAGE_NAME}" --format '{{len .RootFS.Layers}}'
podman image inspect registry.redhat.io/rhel9/rhel-bootc:9.4 --format '{{len .RootFS.Layers}}'

# 3. The Flight Control version you asked for
podman run --rm --entrypoint /bin/sh "${IMAGE_NAME}" -c 'rpm -q flightctl-agent'

# 4. The compose provider you asked for (`podman compose` is a wrapper and
#    reports which external provider it delegates to)
podman run --rm --entrypoint /bin/sh "${IMAGE_NAME}" -c 'podman compose version'
```

## Build the ISO

Must run on aarch64.

```bash
export KICKSTART=./images/rhem.ks
mkdir -p ./output

podman run --rm -it \
  --privileged \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v ./output:/output \
  registry.redhat.io/rhel9/bootc-image-builder:9.4 \
  --local --type iso "${IMAGE_NAME}"

mkksiso ${KICKSTART} ./output/bootiso/install.iso bootc-plus-kickstart.iso
```

## Serve the image from the local registry

Devices pull updates from wherever the image was installed from, so push it to
the local registry before flashing if you want `bootc upgrade` to work on the
LAN. Edit `images/rhem.ks` so the `bootc switch` line points here.

```bash
podman tag "${IMAGE_NAME}" "$HOSTIP:5000/jetson-${COMPOSE_PROVIDER}:${FLIGHTCTL_VERSION}-${DATE_TAG}"
podman push "$HOSTIP:5000/jetson-${COMPOSE_PROVIDER}:${FLIGHTCTL_VERSION}-${DATE_TAG}"
```

## The archive

`images/Containerfile` is the living definition — it floats, and CI pins
`FLIGHTCTL_VERSION` per build so each release matches its tag.

`images/archive/*.Containerfile` are frozen, fully self-contained snapshots of
past images. They are never edited. Copy one out and build it directly if you
need to reproduce an older device image:

```bash
podman build -f images/archive/rhel9.4-flightctl-1.3.0-docker-compose.Containerfile .
```
