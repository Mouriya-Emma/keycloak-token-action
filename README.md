# keycloak-token-action

GitHub composite Action: fetch a short-TTL OIDC access token from Keycloak via the `client_credentials` grant. Standardizes the boilerplate across CI workflows that need to authenticate to internal services (image registry, Komodo Core, etc.) using a Keycloak service account, instead of long-lived basic-auth credentials.

Built for the personal homelab CI/CD setup described in [homelab-tf #123](https://github.com/Mouriya-Emma/homelab-tf/issues/123) (multi-host periphery + Keycloak-backed registry auth RFC). Reusable for any workflow that wants short-TTL bearer tokens via OIDC.

## Usage

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, linux, vctcn]
    steps:
      - uses: actions/checkout@v4

      - id: token
        uses: Mouriya-Emma/keycloak-token-action@v1
        with:
          keycloak-url: https://keycloak.example.com
          realm: my-realm
          client-id: my-service-account
          client-secret: ${{ secrets.KEYCLOAK_CLIENT_SECRET }}

      - name: docker login (registry expects token in password slot)
        run: |
          echo "${{ steps.token.outputs.access-token }}" | docker login registry.example.com -u sa-registry --password-stdin

      - name: build + push
        run: |
          docker buildx build --push -t registry.example.com/my-app:${{ github.sha }} .
```

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `keycloak-url` | yes | — | Keycloak base URL, e.g. `https://keycloak.example.com`. Token endpoint is derived: `${keycloak-url}/realms/${realm}/protocol/openid-connect/token`. |
| `realm` | yes | — | Keycloak realm where the service-account client lives. |
| `client-id` | yes | — | OIDC client ID for the service account. The Keycloak client must be configured with `Client authentication: On` and Service Accounts enabled. |
| `client-secret` | yes | — | Client secret. Pass via repo/org secret; never inline. |
| `scope` | no | `` | OAuth scope(s) to request (space-separated). Empty = realm default scopes. |

## Outputs

| Name | Description |
| --- | --- |
| `access-token` | The `access_token` from Keycloak. Auto-masked in logs via `::add-mask::` before being set as output. |
| `expires-in` | TTL in seconds (from Keycloak's `expires_in` field). Useful for trimming long operations to stay within the token's validity window. |

## Why use this instead of inline `curl`

- **Mask discipline**: the token is registered with GitHub Actions log masking before being exposed as an output. An inline workflow that forgets to do this leaks the token in any subsequent log line.
- **Error surfacing**: on Keycloak error (invalid client, disabled service account, wrong realm), the action prints the response keys (no values) so the failure mode is obvious without leaking the secret-bearing request body.
- **Single contract**: one `keycloak-token-action@v1` reference instead of N copies of "curl POST .../token | jq .access_token" with subtle differences across repos.
- **Pinning**: tag/sha-pin to control upgrade timing (e.g. `keycloak-token-action@v1` for floating major, `@v1.0.0` or `@<sha>` for full immutability).

## Keycloak setup

In Keycloak Admin Console:

1. Create or open a Realm.
2. Clients → Create Client → Client type `OpenID Connect`, give it an ID (e.g. `registry`).
3. Capability config: `Client authentication: On`, `Authentication flow > Service accounts roles: enabled`. Standard flow / Direct access grants can be off (we only use client_credentials).
4. Save → Credentials tab → copy the `Client secret`. This goes into the GitHub repo secret used by `client-secret`.
5. Service accounts roles tab → assign whatever realm/client roles the token bearer needs (e.g. `registry-rw` if your downstream service checks role claims).

## Token TTL

Keycloak's default access-token lifespan is 5 minutes. You can extend per realm (Realm Settings → Tokens → Access Token Lifespan) or per client (Client → Advanced → Access Token Lifespan). Short TTL is the security premise of using OIDC over htpasswd — keep it short, refresh per-job rather than caching across jobs.

## Failure modes

- **HTTP 401 `invalid_client`** — wrong `client-secret` for the named `client-id`. Check the repo secret value.
- **HTTP 400 `unauthorized_client`** — client exists but `Service accounts roles: enabled` is off in Keycloak.
- **HTTP 404** — wrong realm name in `realm` input. Keycloak returns 404 for unknown realms at the OIDC discovery endpoint.
- **`access_token` missing from response** — surfaced as a workflow error with the response keys printed (no values). Usually means Keycloak returned a non-error response shape we don't recognize; file an issue with the response keys list.

## License

MIT.
