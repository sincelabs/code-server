# Coolify

- [Which deployment path do you need?](#which-deployment-path-do-you-need)
- [Before you start](#before-you-start)
- [Path A: deploy the published image with Docker Compose](#path-a-deploy-the-published-image-with-docker-compose)
- [Path B: build a customised image](#path-b-build-a-customised-image)
- [Path C: build from this repository's source](#path-c-build-from-this-repositorys-source)
- [Configuration reference](#configuration-reference)
- [Persistent storage and the uid 1000 trap](#persistent-storage-and-the-uid-1000-trap)
- [Accessing web services you run inside code-server](#accessing-web-services-you-run-inside-code-server)
- [Upgrading](#upgrading)
- [Backups](#backups)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

[Coolify](https://coolify.io) is a self-hosted PaaS. It runs Docker containers on
your own server and puts a Traefik reverse proxy with automatic Let's Encrypt
certificates in front of them. That is a good fit for code-server, which needs
exactly two things from its environment: a TLS-terminating proxy and WebSocket
support. Coolify's proxy provides both with no extra configuration.

## Which deployment path do you need?

Read this section before doing anything else. Picking the wrong path here is the
difference between a two-minute deploy and an hour-long build that runs out of
memory.

| Path                                  | Use it when                                                            | Build time    |
| ------------------------------------- | ---------------------------------------------------------------------- | ------------- |
| **A — published image** (recommended) | You want to run code-server as-is.                                     | none (pull)   |
| **B — customised image**              | You want extra system packages, runtimes, or pre-installed extensions. | 2–5 minutes   |
| **C — build from source**             | You changed code-server's own source or its patches to Code.           | 40–90 minutes |

A note about this repository specifically: `ci/release-image/Dockerfile` (the
Dockerfile that produces the official `codercom/code-server` images) **cannot be
built by Coolify from a fresh clone.** Its first stage is
`COPY release-packages/code-server*.deb /tmp/`, and `release-packages/` is a CI
artifact that does not exist in git. Pointing Coolify at that Dockerfile fails on
that first `COPY`, usually with a `failed to compute cache key` /
`"/release-packages": not found` error. Use one of the three paths below instead.

Files referenced by this guide, all in this repository:

- `coolify/docker-compose.yaml` — path A
- `coolify/Dockerfile` — path B
- `coolify/Dockerfile.source` and `coolify/Dockerfile.source.dockerignore` — path C

## Before you start

1. **A server running Coolify.** code-server itself wants at least 1 GB of RAM
   and 2 vCPUs ([requirements](https://coder.com/docs/code-server/latest/requirements)),
   and you are also running Coolify and Traefik on that machine. 2 GB of RAM is a
   realistic floor; 4 GB is comfortable if you plan to compile anything inside
   the editor. Path C additionally needs ~8 GB of RAM and ~30 GB of free disk on
   whichever machine performs the build.

2. **A domain, pointed at the server, before you deploy.** Create an `A` record
   for `code.example.com` pointing at your server's public IP and wait for it to
   resolve. Let's Encrypt validates over HTTP, so a domain that does not resolve
   yet produces a certificate failure rather than a retry.

   ```console
   $ dig +short code.example.com
   203.0.113.10
   ```

   If you plan to use [port forwarding via subdomains](#accessing-web-services-you-run-inside-code-server),
   add a wildcard `A` record for `*.code.example.com` as well.

3. **Ports 80 and 443 open** to the internet on the server's firewall. Traefik
   needs 80 for the ACME HTTP-01 challenge even though you will only use 443.

4. **A password you have generated, not chosen.** code-server's login page is
   on the public internet and is a single secret away from a shell on your
   server. Generate one:

   ```console
   $ openssl rand -base64 24
   0Nn6r1L9zVqk2QeR7hYtA8sJdF3mXbCw
   ```

## Path A: deploy the published image with Docker Compose

This deploys `codercom/code-server`, the image built from this repository by CI.

### 1. Create the project and resource

1. Open your Coolify dashboard and go to **Projects** → **+ Add**. Name it
   (for example `dev-tools`) and open its **production** environment.
2. Click **+ New** → **Public Repository** (or **Private Repository (GitHub
   App)** if this fork is private and you have connected the GitHub App).
3. Paste the repository URL:

   ```text
   https://github.com/sincelabs/code-server
   ```

4. Set **Branch** to the branch that actually contains `coolify/` — `main` once
   this guide is merged, otherwise the feature branch you are reading it on.
5. Set **Build Pack** to **Docker Compose**.
6. Set **Docker Compose Location** to:

   ```text
   /coolify/docker-compose.yaml
   ```

7. Choose the **Server** and **Destination** (the Docker network) to deploy to,
   then click **Continue**. Coolify parses the compose file and shows the
   `code-server` service.

### 2. Set the domain

In the resource's **General** tab (or **Service Stack** view), find the
**Domains** field belonging to the `code-server` service and enter:

```text
https://code.example.com:8080
```

The `:8080` suffix is not a typo and it does **not** open port 8080 to the
world. In Coolify's Docker Compose build pack, the port appended to the domain
tells the proxy which container port to route to; the site is still served on 443. Omitting it makes Traefik route to port 80 inside the container, where
nothing is listening, which is the single most common cause of a 502 here.

Leave **Force HTTPS** enabled.

### 3. Set the environment variables

Go to the **Environment Variables** tab and add:

| Name                  | Value                      | Notes                                       |
| --------------------- | -------------------------- | ------------------------------------------- |
| `PASSWORD`            | the password you generated | Required. Anyone with it gets a shell.      |
| `CODE_SERVER_VERSION` | `4.135.0`                  | Optional; pin instead of tracking `latest`. |
| `TZ`                  | `Europe/Helsinki`          | Optional.                                   |

Leave **Is Build Variable?** off for all of them — these are runtime values.

If you would rather not store the plaintext password in Coolify, use
`HASHED_PASSWORD` instead. Generate an argon2 hash on any machine with Node:

```console
$ echo -n "0Nn6r1L9zVqk2QeR7hYtA8sJdF3mXbCw" | npx argon2-cli -e
$argon2i$v=19$m=4096,t=3,p=1$wST5QhBgk2lu1ih4DMuxvg$LS1alrVdIWtvZHwnzCM1DUGg+5DTO3Dt1d5v9XtLws4
```

Then add `HASHED_PASSWORD` with **every `$` doubled to `$$`**, because Coolify
feeds the value through Docker Compose variable interpolation:

```text
$$argon2i$$v=19$$m=4096,t=3,p=1$$wST5QhBgk2lu1ih4DMuxvg$$LS1alrVdIWtvZHwnzCM1DUGg+5DTO3Dt1d5v9XtLws4
```

`HASHED_PASSWORD` takes precedence over `PASSWORD`, so set only one.

### 4. Confirm the storage

Open the **Storages** tab. Coolify will have picked up the `code-server-home`
volume declared in the compose file, mounted at `/home/coder`. Do not add extra
mounts at deeper paths yet — read [the uid 1000 trap](#persistent-storage-and-the-uid-1000-trap)
first.

### 5. Deploy

Click **Deploy** and watch **Logs**. A healthy first start looks like this:

```text
[<timestamp>] info  Wrote default config file to /home/coder/.config/code-server/config.yaml
[<timestamp>] info  code-server 4.135.0 <commit>
[<timestamp>] info  Using user-data-dir /home/coder/.local/share/code-server
[<timestamp>] info  Using config file /home/coder/.config/code-server/config.yaml
[<timestamp>] info  HTTP server listening on http://0.0.0.0:8080/
[<timestamp>] info    - Authentication is enabled
[<timestamp>] info      - Using password from $PASSWORD
[<timestamp>] info    - Not serving HTTPS
```

Two lines are worth checking every time:

- **`Using password from $PASSWORD`.** If it instead says
  `Using password from /home/coder/.config/code-server/config.yaml`, your
  environment variable did not reach the container and the effective password is
  a random one written into that file.
- **`Not serving HTTPS`** is correct. Traefik terminates TLS; code-server speaks
  plain HTTP on the internal Docker network.

Open `https://code.example.com`, enter the password, and you are done.

## Path B: build a customised image

Use `coolify/Dockerfile`. It starts from the published release and adds system
packages and extensions, so it builds in minutes rather than an hour.

1. Edit `coolify/Dockerfile` in your fork — the `apt-get install` list and the
   `--install-extension` flags are meant to be changed — then commit and push.
2. In Coolify, create a **Public Repository** / **Private Repository** resource
   as in path A, but set **Build Pack** to **Dockerfile**.
3. Set **Dockerfile Location** to `/coolify/Dockerfile` and leave **Base
   Directory** as `/`.
4. In **General**, set **Ports Exposes** to `8080`. For the Dockerfile build
   pack the port lives here, so the domain field is just
   `https://code.example.com` with **no** `:8080` suffix.
5. Add the same environment variables as path A (minus `CODE_SERVER_VERSION`,
   which is a build arg here — set it under **Build Variables** if you want to
   pin the base image).
6. Under **Storages**, add a **Volume Mount**: name `code-server-home`,
   destination path `/home/coder`.
7. Under **Health Checks**, set the path to `/healthz` and the port to `8080`.
8. **Deploy**.

Two details in that Dockerfile are there to work around traps, and are worth
understanding before you edit it.

**Baked extensions and the home volume.** Installing extensions into
`/home/coder` at build time looks like it works: they are there on the first
deploy, because Docker seeds an empty volume from the image. On every deploy
after that the volume already has contents and shadows the image, so the
extensions you added to the Dockerfile silently vanish. The Dockerfile therefore
installs into `/opt/code-server/extensions` and registers a startup script,
under `ENTRYPOINTD`, that copies that directory into the home extensions
directory the first time it is absent. A fresh volume gets your baked set;
after that, whatever the user installs through the UI is what persists. Adding a
new extension to the Dockerfile later will not retroactively appear in an
existing volume — install it from the UI, or delete the volume.

**Extra flags go in a config file, not in `command:`.** The image's `ENTRYPOINT`
is `["/usr/bin/entrypoint.sh", "--bind-addr", "0.0.0.0:8080", "."]`, already
ending in a positional workspace argument. A Compose `command:` is appended to
that, producing a second positional argument and a startup error. The Dockerfile
writes `/etc/code-server/config.yaml` and points `CODE_SERVER_CONFIG` at it
instead.

## Path C: build from this repository's source

Only take this path if you changed code-server's TypeScript, its `patches/`, or
the pinned Code submodule. It compiles VS Code from scratch.

1. In Coolify, create the resource with **Build Pack** = **Dockerfile** and
   **Dockerfile Location** = `/coolify/Dockerfile.source`.
2. In **General** → **Advanced**, enable **Git Submodules** and **Git LFS**.
   Without submodules, `lib/vscode` is empty and the build stops with an
   explicit error rather than a confusing one.
3. Set **Ports Exposes** to `8080` and the domain to `https://code.example.com`.
4. Raise the **build timeout** if your Coolify version exposes it; the default
   is often shorter than this build.
5. If your Coolify server is small, use a **dedicated build server** (Coolify
   settings → Servers → mark one as a build server). Compiling Code alongside a
   live Traefik on a 2 GB VPS will OOM.
6. Add the storage, environment variables, and health check exactly as in path B,
   then **Deploy**.

`coolify/Dockerfile.source.dockerignore` exists because the repository root
`.dockerignore` excludes everything except `ci/` and `release-packages/`, which
would leave the source build with no source. BuildKit prefers a
`<dockerfile>.dockerignore` file over the root one. Coolify uses BuildKit, but
if you ever build this by hand with BuildKit disabled you will hit the explicit
`.git is missing from the build context` error; the fix is `DOCKER_BUILDKIT=1`.

## Configuration reference

Everything code-server reads from the environment, and what it is for:

| Variable                              | Effect                                                                                                                              |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `PASSWORD`                            | Password for the login page.                                                                                                        |
| `HASHED_PASSWORD`                     | argon2 hash of the password; takes precedence over `PASSWORD`.                                                                      |
| `CODE_SERVER_CONFIG`                  | Path to `config.yaml`. Defaults to `~/.config/code-server/config.yaml`.                                                             |
| `PORT`                                | Overrides the port in `--bind-addr`. Leave it alone here; the entrypoint sets 8080.                                                 |
| `CODE_SERVER_HOST`                    | Overrides the host in `--bind-addr`. Also leave alone.                                                                              |
| `TZ`                                  | Container time zone.                                                                                                                |
| `DOCKER_USER`                         | Renames the `coder` user inside the container.                                                                                      |
| `EXTENSIONS_GALLERY`                  | JSON pointing at a different extension marketplace than Open VSX.                                                                   |
| `CS_DISABLE_FILE_DOWNLOADS`           | `1`/`true` disables downloading files out of the editor.                                                                            |
| `CS_DISABLE_PROXY`                    | `1`/`true` disables the `/proxy` and subdomain port-forwarding routes.                                                              |
| `CS_DISABLE_GETTING_STARTED_OVERRIDE` | `1`/`true` restores the stock Getting Started page.                                                                                 |
| `CODE_SERVER_IDLE_TIMEOUT_SECONDS`    | Shut down after this much inactivity. Must be greater than 60.                                                                      |
| `CODE_SERVER_COOKIE_SUFFIX`           | Suffixes the session cookie. Set this if you run several code-servers on subdomains of one domain, so their cookies stop colliding. |
| `GITHUB_TOKEN`                        | Populates the built-in GitHub authentication.                                                                                       |
| `VSCODE_OPTIONS`                      | Extra arguments passed through to Code.                                                                                             |

For anything that is a CLI flag rather than an environment variable — say
`--disable-telemetry` or `--extensions-dir` — write a config file and point
`CODE_SERVER_CONFIG` at it, as `coolify/Dockerfile` does. Precedence, lowest to
highest: config file, then environment variables, then command-line flags.

One consequence worth knowing: on first start, code-server creates
`~/.config/code-server/config.yaml` containing a randomly generated password.
`PASSWORD` overrides it while it is set, but if you later delete the `PASSWORD`
variable, logins fall back to that random value rather than failing loudly.

## Persistent storage and the uid 1000 trap

The image runs as uid 1000 (`coder`), and this is where most Docker deployments
of code-server go wrong.

When Docker creates an **empty named volume** over a path that **already exists
in the image**, it copies that directory's contents _and its ownership_ into the
volume. `/home/coder` exists in the image and is owned by `coder:coder`, so a
volume mounted there comes out owned by uid 1000 and everything works.

When you mount a volume over a path that does **not** exist in the image —
`/home/coder/project` is the usual choice — Docker creates the mountpoint as
`root:root`. code-server then cannot write to it. If you do this to
`/home/coder/.local/share/code-server`, the container crash-loops on startup.

So: **mount one volume at `/home/coder`.** It covers the workspace, the user
data directory, extensions, `~/.ssh`, and shell history in a single place. That
is what `coolify/docker-compose.yaml` does.

If you genuinely need a separate mount for one project — a bind mount to a
directory on the host, say — fix its ownership on the host first:

```console
$ sudo mkdir -p /data/coolify/code-server/project
$ sudo chown -R 1000:1000 /data/coolify/code-server/project
```

then add it in **Storages** as a bind mount with that source and a destination
under `/home/coder`.

## Accessing web services you run inside code-server

A dev server you start in code-server's terminal on port 3000 is reachable two
ways without touching Coolify:

- **Subpath:** `https://code.example.com/proxy/3000/`. Works immediately. Apps
  that assume they are at the root need `/absproxy/3000/` plus a matching base
  path in their own config; see the [setup guide](./guide.md#accessing-web-services).
- **Subdomain:** `https://3000.code.example.com`. More setup: the wildcard DNS
  record from [Before you start](#before-you-start), `proxy-domain:
code.example.com` in your code-server config file, and — the part Coolify does
  not do for you — a Traefik router that actually sends `*.code.example.com` to
  this container, plus a wildcard certificate, since Let's Encrypt cannot issue
  one over HTTP-01. Expect to add custom Traefik labels and a DNS-01 resolver.
  The subpath route needs none of this; prefer it unless an app truly cannot
  live under a subpath.

Do **not** publish these ports through Coolify's port mappings. They would
bypass code-server's authentication entirely.

## Upgrading

**Path A.** Change `CODE_SERVER_VERSION` to the new tag and click **Redeploy**.
Coolify pulls the new image and recreates the container; the `/home/coder`
volume is untouched, so settings and extensions survive.

**Paths B and C.** Push to the branch Coolify watches. With **Auto Deploy** on,
the push triggers a build; otherwise click **Redeploy**. For path B you also want
to bump the `CODE_SERVER_VERSION` build arg so the base image moves forward.

Check [CHANGELOG.md](../CHANGELOG.md) before upgrading across minor versions.

## Backups

The volume lives on the server at:

```console
$ docker volume ls | grep code-server
$ docker volume inspect <volume-name> --format '{{ .Mountpoint }}'
/var/lib/docker/volumes/code-server-home-<uuid>/_data
```

Back it up while the container is stopped, or accept that open editor state may
be mid-write:

```console
$ sudo tar -czf code-server-home-$(date +%F).tar.gz \
    -C /var/lib/docker/volumes/code-server-home-<uuid>/_data .
```

Coolify's own scheduled-backup feature targets databases, not application
volumes, so this one is on you. A cron job on the host is enough.

## Troubleshooting

**502 Bad Gateway.** Traefik reached the container but nothing answered. In the
Docker Compose build pack, check that the domain ends in `:8080`; in the
Dockerfile build pack, check that **Ports Exposes** is `8080`. Then confirm the
container is actually up in **Logs**.

**The login page rejects the right password.** Look for
`Using password from $PASSWORD` in the logs. If it says the config file instead,
the variable is not reaching the container — a `$` in a `HASHED_PASSWORD` that
was not doubled to `$$` is the usual reason, since Compose interpolation eats it.

**Login succeeds, then bounces back to the login page.** The session cookie is
not surviving. Confirm you are on `https://`, not `http://`. If you host several
code-servers under one parent domain, give each a distinct
`CODE_SERVER_COOKIE_SUFFIX`.

**The editor loads but stays at "Connecting…" or reconnects constantly.** The
WebSocket is not getting through. Traefik handles WebSockets natively, so suspect
what is in front of it: Cloudflare with WebSockets disabled, or Cloudflare SSL
mode set to Flexible (it must be Full or Full (strict)).

**`EACCES: permission denied` in the logs, or the file tree is read-only.** A
volume was mounted at a path that does not exist in the image. See
[the uid 1000 trap](#persistent-storage-and-the-uid-1000-trap).

**The build fails on `COPY release-packages/code-server*.deb`,** typically with
`failed to compute cache key` or `"/release-packages": not found`. You pointed
Coolify at `ci/release-image/Dockerfile`, which can only be built after CI has
produced a `.deb`. Use one of the three paths in this guide.

**Path C build is killed, or fails with `JavaScript heap out of memory`.** The
builder ran out of RAM. Use a dedicated build server with 8 GB or more.

**Path C fails with `lib/vscode is empty`.** **Git Submodules** is off in the
resource's advanced settings.

**Extensions are missing after a redeploy.** They were installed into
`/home/coder` at image build time and the existing volume shadowed them. Install
them at runtime through the UI, or bake them into `/opt` and seed them on
startup as `coolify/Dockerfile` does.

**An extension you want is not in the marketplace.** code-server uses
[Open VSX](https://open-vsx.org), not the Microsoft Marketplace, for
[licensing reasons](./FAQ.md#why-cant-code-server-use-microsofts-extension-marketplace).

## Security notes

code-server gives whoever logs in a terminal running as the container user, on
your server. Treat the deployment accordingly:

- Use a generated password, and prefer `HASHED_PASSWORD` so the plaintext is not
  sitting in Coolify's database.
- Keep **Force HTTPS** on. Password auth over plain HTTP hands the password to
  anyone on the path.
- Do not add port mappings that publish 8080 on the host; that route skips
  Traefik and serves the editor over unencrypted HTTP.
- For anything beyond a personal instance, put an identity-aware proxy in front
  and set `auth: none` behind it, or restrict access by IP at the firewall.
- Set resource limits (**General** → **Resource Limits**) so a runaway build
  inside the editor cannot take down Coolify itself.
- Run one instance per person. code-server has no multi-user model; everyone who
  logs in shares the same files, the same terminal, and the same secrets.
