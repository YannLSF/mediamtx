# Enhanced RTMP multitrack support

This branch contains an experimental MediaMTX extension for Enhanced RTMP
multitrack publishing and forwarding.

It was developed primarily to relay OBS Enhanced Broadcasting streams while
preserving multiple video renditions and RTMP metadata instead of reducing the
input to a single conventional RTMP video track.

## Status

Current MediaMTX base:

- MediaMTX: `v1.21.1`
- custom branch: `enhanced-rtmp-multitrack`
- MediaMTX patch commit:
  `dd9f4dfb1c51d074db686c53767c2e0c31c991d1`

The RTMP protocol changes are provided by a companion gortmplib fork:

- repository: https://github.com/YannLSF/gortmplib
- branch: `enhanced-rtmp-multitrack`
- pinned commit:
  `3f7e4ab7ed7de6f2fdd5c53ab0315fb5c9f43d45`

The gortmplib fork is included as the `gortmplib-local` Git submodule and is
referenced from `go.mod` with:

```go
replace github.com/bluenviron/gortmplib => ./gortmplib-local
```

The Git submodule commit is authoritative. The branch configured in
`.gitmodules` is provided for convenience when working on the companion fork.

## What this changes

The patch extends the RTMP path through MediaMTX so that information required
by Enhanced RTMP multitrack streams can survive publishing, storage in the
MediaMTX stream object, and subsequent RTMP forwarding.

At a high level, the patch adds:

- Enhanced RTMP multitrack support through the companion gortmplib fork;
- preservation of RTMP metadata received from a publisher;
- propagation of that metadata to RTMP readers and forward destinations;
- forwarding support that retains the original request information required
  by dynamic destination templates;
- support for `$MTX_QUERY` in the forwarding path used by the included example;
- Docker build support for the patched source tree.

The companion gortmplib changes include:

- Enhanced RTMP multitrack message handling;
- video track IDs;
- key-frame state for multitrack video messages;
- multitrack publisher metadata;
- publisher FourCC advertisement for AVC, HEVC and MPEG-4 Audio;
- graceful publisher shutdown with `FCUnpublish` and `deleteStream`.

This is intentionally a focused extension rather than a redesign of the
MediaMTX RTMP stack.

## Clone

Clone the branch together with its submodule:

```sh
git clone \
  --recurse-submodules \
  --branch enhanced-rtmp-multitrack \
  https://github.com/YannLSF/mediamtx.git

cd mediamtx
```

If the repository was cloned without `--recurse-submodules`, initialize it
afterwards:

```sh
git submodule update --init --recursive
```

You can verify the exact gortmplib revision with:

```sh
git submodule status gortmplib-local
git -C gortmplib-local rev-parse HEAD
```

The expected revision for this MediaMTX commit is:

```text
3f7e4ab7ed7de6f2fdd5c53ab0315fb5c9f43d45
```

Do not use `git submodule update --remote` unless you intentionally want to
move the submodule away from the commit pinned by MediaMTX.

## Build

A dedicated Dockerfile is included:

```sh
docker build \
  -f Dockerfile.twitch \
  -t mediamtx:twitch-eb-1.21.1 \
  .
```

The current Docker build uses:

- `golang:1.26-alpine3.24` for compilation;
- `alpine:3.22` for the runtime image.

The build performs `go generate ./...` followed by a static MediaMTX build.

A clean clone with `--recurse-submodules` has been verified to build
successfully with this Dockerfile.

## Example configuration

`mediamtx-patched-test.yml` is a minimal test configuration for the patched
RTMP forwarding path.

It enables RTMP on port `1938`, disables the other media protocols, and uses a
regular-expression path:

```yaml
paths:
  "~^app/(.+)$":
    forward:
      - dest: "rtmps://ingest.global-contribute.live-video.net/app#$G1?$MTX_QUERY"
```

In this example:

- `$G1` contains the first regular-expression capture group;
- `$MTX_QUERY` preserves the original RTMP query string when constructing the
  forwarding destination.

The Twitch endpoint in this file is an example target. Review and adapt the
configuration before using it in production.

Never commit real stream keys, authentication tokens or other credentials to
the repository.

## Architecture

The relevant flow is:

```text
OBS / Enhanced RTMP publisher
            |
            v
     patched gortmplib
            |
            v
      MediaMTX RTMP
            |
            +---- multiple media tracks
            |
            +---- RTMP metadata
            |
            v
       MediaMTX Stream
            |
            v
      RTMP forwarding
            |
            v
  Enhanced RTMP destination
```

When an RTMP publisher connects, MediaMTX stores the metadata returned by
gortmplib together with the stream.

That metadata is then supplied to the RTMP output path so that forwarding does
not reconstruct the stream solely from the conventional MediaMTX media
description.

## Source changes

The MediaMTX-specific changes are currently concentrated in:

```text
internal/core/path.go
internal/defs/path.go
internal/forward/dest_handler.go
internal/forward/manager.go
internal/forward/rtmp/dest.go
internal/protocols/rtmp/from_stream.go
internal/servers/rtmp/conn.go
internal/stream/stream.go
```

Additional integration files are:

```text
.gitmodules
Dockerfile.twitch
go.mod
gortmplib-local
mediamtx-patched-test.yml
```

The main MediaMTX stream structure contains an additional `RTMPMetadata` value.
The RTMP publisher stores the metadata obtained from gortmplib, and the RTMP
reader/forwarding side receives it again when creating the outgoing stream.

## Companion gortmplib fork

The protocol-level changes deliberately live in a separate gortmplib fork
instead of being copied into the MediaMTX tree.

The MediaMTX repository therefore contains a Git submodule:

```text
gortmplib-local
```

and `go.mod` redirects the normal dependency to this local checkout.

This arrangement has two useful properties:

1. MediaMTX and gortmplib changes remain independently reviewable.
2. A MediaMTX commit pins an exact gortmplib commit, making a recursive clone
   reproducible.

## Updating to a newer MediaMTX release

Do not blindly rebase this branch onto a new MediaMTX release. MediaMTX and
gortmplib APIs may have changed.

A safer update procedure is:

1. fetch the latest upstream MediaMTX history;
2. determine which upstream gortmplib version the target MediaMTX release uses;
3. update the gortmplib fork first and port the Enhanced RTMP changes onto the
   corresponding upstream gortmplib revision;
4. run the complete gortmplib test suite;
5. create a new MediaMTX branch from the desired upstream MediaMTX tag;
6. reapply or port the MediaMTX Enhanced RTMP changes;
7. update `gortmplib-local` to the newly tested gortmplib commit;
8. verify the `replace` directive in `go.mod`;
9. build MediaMTX;
10. test publishing, multitrack forwarding and publisher shutdown;
11. perform a clean recursive clone and build it again before publishing.

Useful references for comparing the current patch are:

```sh
git diff v1.21.1..enhanced-rtmp-multitrack

git -C gortmplib-local diff \
  v1.0.3..enhanced-rtmp-multitrack
```

When resolving conflicts, preserve behavior rather than mechanically preserving
the old patch lines.

## Development checks

Before committing MediaMTX changes:

```sh
git diff --check
git status --short
git submodule status
```

For gortmplib, its full test suite can be run without installing Go locally:

```sh
cd gortmplib-local

docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e GOCACHE=/tmp/go-cache \
  -e GOMODCACHE=/tmp/go-mod-cache \
  -v "$PWD:/src" \
  -w /src \
  golang:1.26-alpine \
  go test ./...
```

Return to the MediaMTX repository afterwards:

```sh
cd ..
```

For MediaMTX, the final reproducibility check is a Docker build from a fresh
recursive clone.

## Upstream

This repository is a fork of:

https://github.com/bluenviron/mediamtx

The Enhanced RTMP work in this branch is maintained separately from upstream
MediaMTX and should not be assumed to be supported by the upstream project.
