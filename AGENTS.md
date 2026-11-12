# AGENTS.md — openwebui (Teva)

Open WebUI at **https://chat.teva.services**, deployed by Komodo as the
Stack **`openwebui`** on server **`docker01`** (AWS `i-08e072fa4421df904`,
us-east-2, arm64). The control plane is `TevaServices/komodo`; this repo is
`TevaServices/openwebui`.

Read the control plane's `AGENTS.md` before changing the host: SSM-only access,
Komodo deploy mechanics, and the secrets policy are all documented there.

## Boundaries

- **One public name:** `chat.teva.services`, served by the NetBird reverse
  proxy on the `netbird` host. The proxy target is `docker01` (`100.91.204.199`)
  port `8080`.
- **No `cloudflared` network.** This deployment uses the NetBird proxy, not the
  homelab Cloudflare tunnel.
- **Loopback only inside the host.** The container publishes
  `127.0.0.1:8080`; `openwebui-forwarder` binds `100.91.204.199:8080` and
  forwards to loopback. Do not publish the container on `0.0.0.0` or the VPC.
- **The provider is the hermes stack.** `OPENAI_API_BASE_URL` is
  `http://litellm:4000/v1` on `hermes-net`; `ENABLE_OLLAMA_API=false`.
- **SSO is Zitadel.** Local login is disabled. The OIDC app redirect is
  `https://chat.teva.services/oauth/callback`; the domain fence is
  `tevaservices.com`. The app lives in the Teva Zitadel org.
- **State is the pinned volume.** `open-webui-data` is explicitly
  `openwebui_open-webui-data`; renaming it starts chat history empty. Do not
  change the explicit `name:`.

## Secrets

`/etc/hermes/openwebui.env` on `docker01` (root-owned, mode 600) supplies:

| Variable | What it is |
|---|---|
| `OPENAI_API_KEY` | LiteLLM virtual key scoped to `all-proxy-models`; never the master key |
| `FIRECRAWL_API_KEY` | Must equal the hermes stack's `firecrawl.env` `TEST_API_KEY` |
| `OAUTH_CLIENT_ID` | Teva Open WebUI Zitadel application client id |
| `OAUTH_CLIENT_SECRET` | That application's secret |
| `WEBUI_SECRET_KEY` | Open WebUI session/signing secret; preserve it with the volume |

The stack environment sets `OPENWEBUI_ENV_FILE=/etc/hermes/openwebui.env`.
Nothing secret is in git.

## Deploy and verify

Komodo deploys the stack from `/etc/komodo/stacks/openwebui`. For a manual
break-glass run:

```bash
cd /etc/komodo/stacks/openwebui
docker compose -p openwebui --env-file /etc/hermes/openwebui.env up -d
```

Verify:

```bash
docker ps --filter name=open-webui --format '{{.Status}}'
curl -sS -o /dev/null -w '%{http_code}\n' https://chat.teva.services
docker logs --tail=50 open-webui
```

A healthy gateway check:

```bash
docker exec open-webui python3 -c 'import urllib.request as u, os; print(u.urlopen(u.Request("http://litellm:4000/v1/models", headers={"Authorization": "Bearer " + os.environ["OPENAI_API_KEY"]})).status)'
```

## Rollback

The old `bcross/openwebui` deployment is untouched. To roll back, point the
NetBird `chat.teva.services` route back to the old host and stop this stack.
Do not delete `openwebui_open-webui-data`.
