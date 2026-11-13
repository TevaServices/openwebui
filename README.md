# Open WebUI — Teva chat front door

[Open WebUI](https://github.com/open-webui/open-webui) for Teva staff at
**https://chat.teva.services**. Sign-in is Zitadel SSO for `@tevaservices.com`
accounts; local login is disabled. Every model request goes through the
`hermes` stack's LiteLLM gateway on the shared `hermes-net` Docker network.

Deployed as the Komodo Stack **`openwebui`** on server **`docker01`** (AWS
`i-08e072fa4421df904`, us-east-2). The repo is `TevaServices/openwebui`.

## What is pinned

- `open-webui-data` is explicitly named `openwebui_open-webui-data`. Do not
  rename it; that is the existing chat/settings volume.
- The LiteLLM virtual key is a Komodo/host secret, never git. It is scoped to
  the gateway catalogue, not the LiteLLM master key.
- The container is published on host loopback only. `openwebui-forwarder`
  binds the host's NetBird address (`100.91.204.199:8080`) and forwards to
  loopback, so the NetBird reverse proxy can reach it while the VPC/internet
  cannot.
- `OPENWEBUI_ENV_FILE` points at `/etc/hermes/openwebui.env` on the target
  host. That file holds `OPENAI_API_KEY`, `FIRECRAWL_API_KEY`,
  `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, and `WEBUI_SECRET_KEY`.

## Compose

```bash
docker compose -p openwebui --env-file /etc/hermes/openwebui.env up -d
```

Komodo runs the same project from `/etc/komodo/stacks/openwebui` with
`OPENWEBUI_ENV_FILE=/etc/hermes/openwebui.env` in the stack environment.

## Verify

```bash
docker ps --filter name=open-webui --format '{{.Status}}'
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080
curl -sS -o /dev/null -w '%{http_code}\n' https://chat.teva.services
```

The gateway is healthy when:

```bash
docker exec open-webui python3 -c 'import urllib.request as u, os; print(u.urlopen(u.Request("http://litellm:4000/v1/models", headers={"Authorization": "Bearer " + os.environ["OPENAI_API_KEY"]})).status)'
```

prints `200`.

## SSO

- Issuer: `https://login.tevaservices.com`
- Discovery: `https://login.tevaservices.com/.well-known/openid-configuration`
- Redirect URI: `https://chat.teva.services/oauth/oidc/callback`
- Domain fence: `OAUTH_ALLOWED_DOMAINS=tevaservices.com`
- Local login: disabled (`ENABLE_LOGIN_FORM=false`)
- `ADMIN_EMAIL` seeds the first administrator.

The `Teva Open WebUI` Zitadel project/application is the OIDC client. The
client id is public; the secret lives in `/etc/hermes/openwebui.env`.

## Web search

Web search uses the self-hosted Firecrawl through `firecrawl-guard:3003` on
`hermes-net`. `FIRECRAWL_API_KEY` must equal the hermes stack's
`firecrawl.env` `TEST_API_KEY`.
