# podmania

Minimal container images for self-hosted applications, designed for rootless Podman.

Built with [Nix](https://nixos.org/) and [nix2container](https://github.com/nlewo/nix2container).

## Why

### The rootless Podman problem

[Rootless Podman](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md) runs containers as an unprivileged user, which is the most secure way to run containers on a single host. But most popular images are not built for it.

Images from [LinuxServer.io](https://fleet.linuxserver.io/) and similar projects ship a full s6-overlay init that starts as root inside the container,
then switches to the user specified by `PUID`/`PGID`.

Rootless Podman can map container root to your host user, but the container still has no real UID 0 privileges.

When s6 tries to `setuid` from root to the `PUID` user, the kernel rejects it because an unprivileged process can't change its UID.
The init either crashes or silently skips the user switch and runs everything as the wrong user.

### `userns_mode: "keep-id"`

Podman's
[`userns_mode: "keep-id"`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#userns-mode) maps your host UID directly into the container with no offset.

Since there's no UID remapping needed at runtime, there's no need for s6-overlay or `PUID`/`PGID` variables.

The catch: the image has to work without an init.

The first process must be able to start directly as an arbitrary unprivileged UID and run without ever calling `chown`, `useradd`, `chmod`, or anything that assumes UID 0.

### Podmania images are different

- **The entrypoint is the application binary.**
  There is no init system, no supervisor, and no startup script.
  The container runs one process.
  
- **Nothing assumes root.**
  There are no `chown` calls, no `useradd`, no `mkdir` with ownership changes.
  The app starts as the rootless host user Podman maps in.
  
- **The image is distroless.**
  It contains the application binary and its `/nix/store` runtime dependencies.
  There is no shell, no `apt`, and no operating system.
  
- **Builds are reproducible.**
  Images are built with [Nix](https://nixos.org/) and
  [nix2container](https://github.com/nlewo/nix2container).
  The same flake inputs produce the same image hash every time.

## How it works

Every image inherits from a shared [base image](https://github.com/podmania/base) that provides CA certificates, timezone data, and a minimal `/etc/passwd` (NSS) setup.

Each application image layers only its package on top of that.

A compose file looks like this:

```yaml
services:
  opencloud:
    image: ghcr.io/podmania/opencloud:latest
    userns_mode: "keep-id"
    container_name: opencloud
    ports:
      - 9200:9200
    volumes:
      - ./config:/etc/opencloud
      - ./data:/var/lib/opencloud
      - /etc/localtime:/etc/localtime:ro
    environment:
      - OC_URL=https://localhost:9200
    restart: unless-stopped
```

## Docker

While these images are primarily designed for rootless podman, they should work fine in docker as well. The only difference is that docker does not support the
`userns_mode: "keep-id"` flag. Instead, you'll need to use `user: "1000":"1000"` (replace 1000 with your host user UID/GID)

Example:
```yaml
services:
  opencloud:
    image: ghcr.io/podmania/opencloud:latest
    user: "1000:1000"
    container_name: opencloud
    ports:
      - 9200:9200
    volumes:
      - ./config:/etc/opencloud
      - ./data:/var/lib/opencloud
      - /etc/localtime:/etc/localtime:ro
    environment:
      - OC_URL=https://localhost:9200
    restart: unless-stopped
```
