# tail-hosted-opengist

[Opengist](https://opengist.io/) hosted privately on a Tailscale tailnet.

Login uses OAuth2 against a [tail-hosted-gitea](https://github.com/mikynov/tail-hosted-gitea) instance, which must be set up first.

## Setup

1. Create `.env`:
   ```bash
   OG_EXTERNAL_URL=https://gist.{{tailnet}}.ts.net
   OG_HTTP_PORT=6157
   OG_SSH_PORT=2222
   OG_GITEA_URL=https://gitea.{{tailnet}}.ts.net/
   ```
   Using the same value for `OG_SSH_PORT` and `GITEA_SSH_PORT` will make your life easier.
2. Create a Tailscale auth key and store it in `secrets/tailscale_authkey`.
   * [Definition -> Tags](https://console.tailscale.com/admin/acls/visual/tags) -> Create tag
      * Tag name: **gist**
      * Tag owner: leave empty, see [Define tag owners](https://tailscale.com/docs/features/tags#define-tag-owners)
      * Description: **Opengist**
   * [Settings -> Keys](https://console.tailscale.com/admin/settings/keys) -> Generate auth key...
      * Description: **Opengist**
      * Reusable: **yes**
      * Expiration: **90**
      * Tags: **tag:gist**
3. Generate Opengist secrets:
   ```bash
   ./run generate_secrets
   ```
   This will add `OG_SECRET_KEY` to `.env`
4. Make sure Gitea is running (`./run up serve` in [tail-hosted-gitea](https://github.com/mikynov/tail-hosted-gitea)) and **log in as admin**
5. Create OAuth2 application at `https://gitea.{{tailnet}}.ts.net/-/admin/applications`
   * Application Name: **Opengist**
   * Redirect URI: `https://gist.{{tailnet}}.ts.net/oauth/gitea/callback`
   * Confidential Client: **true**
6. Create `secrets/opengist_secrets.env` with the Gitea OAuth2 secrets:
   ```bash
   OG_GITEA_CLIENT_KEY=<Client-ID-from-Gitea>
   OG_GITEA_SECRET=<Client-Secret-from-Gitea>
   ```
7. Setup access in Tailscale, see [Access controls](#access-controls)
8. Start containers and expose via [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve)
   ```bash
   ./run up serve
   ```
9. Click "_Continue with Gitea@MyMachine account_" to create the admin user using OAuth2
10. After logging in, go to Admin -> Configuration (`https://gist.{{tailnet}}.ts.net/-/admin-panel/configuration`)
    * Disable signup: **Off**
    * Disable login form: **On**

    As a result, anyone can register in Gitea and then log in to Opengist through Gitea's OAuth2, but registration and login through Opengist's own forms are disabled.

    For enhanced privacy, consider the following settings:
    * Require login: **On**
    * Disable Gravatar: **On**

## Notes

- `config.yml` holds only non-default values.
- Opengist reaches Gitea (OAuth2) through the sidecar's outbound HTTP proxy (`HTTPS_PROXY=http://127.0.0.1:1056`).
- Runtime state (`tailscale-state/`, `data/*` except `data/custom/`) is git-ignored.
- For debugging purposes, uncomment the `ports` of the Tailscale sidecar to also expose Opengist on http://localhost:6157.

## Tailscale

### Access controls

Complements the [Gitea access controls](https://github.com/mikynov/tail-hosted-gitea#access-controls), which also explain how to remove the default "_allow all_" rule.

1. Allow Opengist traffic

   Add **tag:gist** to _Destination_ in "_Gitea_" rule (created in [Gitea access controls](https://github.com/mikynov/tail-hosted-gitea#access-controls))

   [Policies -> General access rules](https://console.tailscale.com/admin/acls/visual/general-access-rules) -> "_Gitea_" rule -> `...` -> Edit

   * Source: **autogroup:member, autogroup:shared**
   * Destination: **tag:gitea, tag:gist**
   * Port and protocol: **tcp:443, tcp:2222** (HTTPS and `OG_SSH_PORT`)
   * Description: **Gitea and Opengist**

2. Allow Opengist to access Gitea for OAuth2

   [Policies -> General access rules](https://console.tailscale.com/admin/acls/visual/general-access-rules) -> Add rule

   * Source: **tag:gist**
   * Destination: **tag:gitea**
   * Port and protocol: **tcp:443**
   * Description: **Opengist-Gitea OAuth2**

3. Optionally, verify in [JSON editor](https://console.tailscale.com/admin/acls/file)

   Besides other settings, there should be something like:
   ```jsonc
   "tagOwners": {
      // Gitea
      "tag:gitea": [],
      // Opengist
      "tag:gist": []
   },
   "grants": [
      // Gitea and Opengist
      {
         "src": ["autogroup:member", "autogroup:shared"],
         "dst": ["tag:gitea", "tag:gist"],
         "ip": ["tcp:443", "tcp:2222"]
      },
      // Opengist-Gitea OAuth2
      {
         "src": ["tag:gist"],
         "dst": ["tag:gitea"],
         "ip": ["tcp:443"]
      },
   ],
   ```
   and no grant with `"src": ["*"]`

### Sharing

Share the machines as described in the [Gitea README](https://github.com/mikynov/tail-hosted-gitea#sharing). Share **both** Gitea and Opengist. Logging in sends the browser to Gitea (for OAuth2), so sharing only Opengist is not enough.
