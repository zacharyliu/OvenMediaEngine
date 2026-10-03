# Patched OvenMediaEngine image

This branch starts from upstream v0.21.0 (a35b09d11a82035f63f1d22e3fec9bbadb8e0aec).
The runtime source change lowers RTP_DEFAULT_MAX_PACKET_SIZE from 1472 to 1200
in src/modules/rtp_rtcp/rtp_packet.h. The smaller packet budget avoids the
fragmentation observed on a Fly Machine with a 1420-byte interface MTU. RTMP
publishers do not need encoder slice-size settings or video transcoding.
Occasional unfragmented UDP loss and brief freezes remained in desktop tests;
the change does not establish a reliability guarantee. RTX/NACK remains useful.

The root Dockerfile uses the upstream build and runtime stages. Its only build
change adds BUILD_JOBS (default 2) to limit compilation parallelism. Dependencies
are built by upstream CMake rather than copied from an existing OME image.

## Publishing

The Publish patched OME image workflow builds Linux AMD64 and publishes:

- ghcr.io/zacharyliu/ovenmediaengine:v0.21.0-rtp1200
- ghcr.io/zacharyliu/ovenmediaengine:sha-<full-commit-sha>

It uses USE_LOCAL=true so the Dockerfile builds this checkout, including the
patch, instead of cloning unmodified upstream source. GitHub Actions supplies
GITHUB_TOKEN for publishing; no Docker Hub token or personal access token is
needed. The executable is smoke-tested before either image tag is pushed.
Action versions are pinned by commit. The package must be public for anonymous
pulls from Fly or Docker; GitHub initially creates container packages as private.

Run the workflow manually to rebuild, or push a source/build change on this
branch. The version tag can move; deploy by registry digest to select an exact
image. Preserve the fork and commit when distributing the image so its modified
source remains available with the upstream license.

## Building locally

```sh
docker build --platform linux/amd64 --build-arg USE_LOCAL=true \
  --build-arg BUILD_JOBS=6 -t ovenmediaengine:rtp1200 .
```

This builds the same upstream Dockerfile, including dependencies. A cold build
can take substantially longer than the earlier experiment that reused libraries
from the official runtime image.

## Deployment image

Keep application settings in a separate, small deployment directory:

```dockerfile
FROM ghcr.io/zacharyliu/ovenmediaengine@sha256:<published-digest>
COPY Server.xml /opt/ovenmediaengine/bin/origin_conf/Server.xml
```

The inherited command launches OME in origin mode. For a colocated test player,
also copy its HTML, a small HTTP server and a startup script. fly.toml stays in
the deployment directory and defines Machine resources and public ports.
Neither deployment configuration nor the player belongs in the reusable base
image. Deployments consume the published image and do not compile OME again.
