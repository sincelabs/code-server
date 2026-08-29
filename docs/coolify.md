# Deploy on Coolify

Runs code-server behind Coolify's proxy with HTTPS. About 10 minutes.

Replace `code.example.com` with your domain throughout.

## 1. Point your domain at the server

Add an `A` record for `code.example.com` → your Coolify server's IP, then check it:

```console
$ dig +short code.example.com
203.0.113.10
```

Do this first. Let's Encrypt validates over HTTP and fails if the domain does not resolve yet.

## 2. Generate a password

```console
$ openssl rand -base64 24
```

Save it. This is the login for the editor, and the editor is a shell on your server.

## 3. Create the resource

In Coolify:

- **Projects** → **+ Add** → open the **production** environment
- **+ New** → **Public Repository**
- Repository: `https://github.com/sincelabs/code-server`
- Branch: `main`
- Build Pack: **Docker Compose**
- Docker Compose Location: `/coolify/docker-compose.yaml`
- Pick your server, then **Continue**

## 4. Set the domain

In the `code-server` service's **Domains** field, enter:

```text
https://code.example.com:8080
```

Keep the `:8080`. It tells the proxy which container port to route to — it does not open 8080 to the internet. Leave it off and you get a 502.

Leave **Force HTTPS** on.

## 5. Set the password

**Environment Variables** → add `PASSWORD`, with the value from step 2.

## 6. Deploy

Click **Deploy**, then watch **Logs** for:

```text
HTTP server listening on http://0.0.0.0:8080/
  - Authentication is enabled
    - Using password from $PASSWORD
```

## 7. Log in

Open `https://code.example.com` and enter the password.

## If it breaks

| Symptom                     | Fix                                                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 502 Bad Gateway             | The domain is missing `:8080` (step 4).                                                                                                           |
| Password rejected           | Logs say `Using password from /home/coder/.config/...` instead of `$PASSWORD`. The variable never reached the container — re-add it and redeploy. |
| Stuck on "Connecting…"      | WebSockets blocked upstream. On Cloudflare: turn WebSockets on, and set SSL mode to Full, not Flexible.                                           |
| `EACCES: permission denied` | You added a volume at a path like `/home/coder/project`. Remove it — the compose file already persists all of `/home/coder`.                      |

## Day-two notes

**Upgrading.** Add `CODE_SERVER_VERSION` as an environment variable (for example `4.135.0`), change it, and **Redeploy**. Everything under `/home/coder` survives.

**Backups.** One Docker volume holds settings, extensions, and your files:

```console
$ docker volume inspect "$(docker volume ls -q | grep code-server)" --format '{{ .Mountpoint }}'
$ sudo tar -czf code-server-backup.tar.gz -C <that path> .
```

**Do not add port mappings.** Publishing 8080 on the host serves the editor over plain HTTP and skips the proxy's TLS.

**Extensions** come from [Open VSX](https://open-vsx.org), not Microsoft's marketplace.

## Custom images

Only if you need more than the stock editor. Both use build pack **Dockerfile**, **Ports Exposes** `8080`, a domain with **no** `:8080` suffix, and a **Volume Mount** at `/home/coder` under **Storages**.

| Need                                       | Dockerfile Location          | Extra settings                                                                  |
| ------------------------------------------ | ---------------------------- | ------------------------------------------------------------------------------- |
| Extra packages or pre-installed extensions | `/coolify/Dockerfile`        | none                                                                            |
| You changed code-server's own source       | `/coolify/Dockerfile.source` | Enable **Git Submodules** and **Git LFS**. Compiles VS Code: ~1 hour, 8 GB RAM. |

`ci/release-image/Dockerfile` cannot be built by Coolify — it copies a `.deb` that only CI produces.
