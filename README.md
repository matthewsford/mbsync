# mbsync-image

Minimal Alpine image that runs [mbsync](https://isync.sourceforge.io/) (isync) as a non-root user
(UID/GID 1002), for mirroring one IMAP mailbox into another. Built for the `stalwart` namespace in
the `server-ops` cluster.

## How it behaves

- The entrypoint is `mbsync` under `tini`; container args are passed straight to it, e.g.
  `--config /etc/mbsync/mbsyncrc camoh`.
- `HOME` is `/tmp`, so the container works with `readOnlyRootFilesystem` as long as `/tmp` is a
  writable volume. mbsync keeps its sync state wherever the config's `SyncState` points, so mount
  a persistent volume there.
- mbsync does not require its config file to be owned by the running user, so a mounted Secret
  can be used directly.

## License

The files in this repository are licensed under `GPL-2.0-or-later`, the same base license as
isync (see [LICENSES/](LICENSES/GPL-2.0-or-later.txt) and `REUSE.toml`; check compliance with
`reuse lint`). The image also contains isync, Alpine Linux and their dependencies, which keep
their own licenses.

## Build

```sh
docker build -t mbsync-image .
```

### Updating the base image

The base image is pinned by digest; the `apk` packages are unpinned and resolve to the latest
version in Alpine 3.22 at build time. To bump the base digest:

1. `docker buildx imagetools inspect alpine:3.22` and copy the `Digest:` line.
2. Update the `FROM` line in the `Dockerfile` and rebuild.

GitHub Actions publishes `ghcr.io/<owner>/mbsync-image` on pushes to `main` (tags `latest` and
`YYYY.MM.DD-<sha>`), on `v*` tags, and on a weekly schedule for base-image updates. Pull requests
build without pushing.

## Using it in Kubernetes

Pin the image by digest in the CronJob. The container needs:

- the config mounted read-only, e.g. at `/etc/mbsync/mbsyncrc`
- a writable `/tmp` (an `emptyDir`) and a persistent volume for `SyncState`
- `runAsUser: 1002`

Example config (copy new mail from a remote IMAP account into another, never deleting anything):

```
IMAPAccount source
Host imap.example.com
User "me@example.com"
Pass "app-password"
TLSType IMAPS

IMAPStore source-remote
Account source

IMAPAccount dest
Host mail.example.net
User "me@example.net"
Pass "app-password"
TLSType IMAPS

IMAPStore dest-local
Account dest

Channel mirror
Far :source-remote:
Near :dest-local:
Patterns *
Create Near
Remove None
Expunge None
Sync Pull
SyncState /state/
```
