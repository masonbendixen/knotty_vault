---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/13/2026
Version: 0.1
tags: 
---
# Overview

I have been currently running ng serve with just HTTP with a development mode in angular with a proxy.conf.json that points to the C++ server running on the same machine. To angular, /api calls are routed to this other server on the same machine so there are no cross site security issues. As I advance in the project we are hitting a point where I need to address a more complicated security solution.

I'm currently trying to host the Square Web Payments and am getting:
```json
{
    "detail": "CURL error: SSL peer certificate or SSH remote key was not OK",
    "status": 400,
    "title": "Validation Error",
    "type": "validation_error"
}
```
When trying to use Web Payments. This is apparently an SSL certificate verification error from the C++ backend. Apparently libcurl on Windows can't find a CA certificate bundle to verify Square's SSL certificate. I need to solve this and would like something that is checked into the tree and will just work when anyone enlists in the source tree and runs the server.

I also am not sure if I can have ng serve run so that it serves up with HTTPS. I'd like to figure out if this is possible.

I will also need to be able to set things up for development mode where the C++ web server is exposed externally via HTTPS so that the Square callback features can call to endpoints on it. I have read that I can do this with ngrok but I want to detail that here.

I also eventually need to be able to run the server for real with HTTPS. Eventually, the C++ Web Server will be hosted on AWS and will serve the Angular app and the APIs. Flipping things from how they are now. I don't need to solve this now but want to track it and have some idea what will be involved in doing that.

I'd like to enter plan mode and have you use this document to create sections for all these scenarios and start with a high level plan for each. Please leave this overview alone but replace the sections below with sections to accomplish what is listed in this overview. Please use this document as your MD file for your plan mode. Please use this as your scratch pad to place notes and list open questions for me to answer. We will work on it section by section iterating to improve the plan and then I will work with you after each plan is complete to implement each plan but I will drive this.
# Current State

| Component | Protocol | SSL/TLS Status |
|-----------|----------|----------------|
| Angular Dev Server (`ng serve`) | HTTP | No HTTPS configured |
| Angular → C++ Proxy | HTTP | `secure: false` in `proxy.conf.json` |
| C++ Crow Web Server | HTTP | No SSL context configured |
| C++ HttpClient (libcurl → Square API) | HTTPS | **No CA bundle — BROKEN** |
| Email (SMTP → Gmail) | SMTP+SSL | Working (port 465, mailio handles certs) |
| Square Web Payments SDK (browser) | HTTPS | Working (browser handles certs) |

**Key files**:
- `server/knottyyoga_server/src/util/http/http_client.cpp` — libcurl wrapper, no SSL options set
- `server/knottyyoga_server/src/util/http/http_client.h` — `HttpClient` interface
- `server/knottyyoga_server/conanfile.py` — Has libcurl 7.86.0 + OpenSSL 3.5.2 as dependencies
- `ui/src/proxy.conf.json` — Angular dev proxy to `http://127.0.0.1:18080`
- `ui/angular.json` — Build/serve configurations
- `server/knottyyoga_server/src/main.cpp` — Crow server startup, HTTP only

---

# Section 1: Fix SSL for Outgoing HTTPS Calls (libcurl → Square)

## Problem

The C++ `HttpClientImpl` in `http_client.cpp` calls `curl_easy_perform()` without setting any SSL options. On Windows, libcurl (built with OpenSSL via Conan) has no default CA certificate bundle, so HTTPS calls to `https://connect.squareupsandbox.com` fail with `CURLE_SSL_PEER_CERTIFICATE`.

On Linux (Docker container), this may work because the OS provides `/etc/ssl/certs/ca-certificates.crt`, but relying on this is fragile.

## Solution: Check in Mozilla CA Bundle

1. **Download Mozilla's `cacert.pem`** from https://curl.se/ca/cacert.pem and check it into the repo at `server/knottyyoga_server/certs/cacert.pem`
2. **Add `CURLOPT_CAINFO`** to `http_client.cpp` pointing to this file
3. **Make the path configurable** via environment variable `CA_BUNDLE_PATH` with a fallback to a path relative to the working directory

## Implementation Steps

### 1.1: Add CA bundle to repo
- Download `cacert.pem` from `https://curl.se/ca/cacert.pem`
- Save to `server/knottyyoga_server/certs/cacert.pem`
- Add a `certs/README.md` explaining what this file is and how to update it

### 1.2: Modify `HttpClientImpl::Execute()` in `http_client.cpp`
Add SSL configuration before `curl_easy_perform()`:

```cpp
// SSL certificate verification
curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 1L);
curl_easy_setopt(curl, CURLOPT_SSL_VERIFYHOST, 2L);

// Use CA bundle - check CURL_CA_BUNDLE env var first, then fall back to relative path
const char* caBundle = getenv("CURL_CA_BUNDLE");
if (caBundle) {
    curl_easy_setopt(curl, CURLOPT_CAINFO, caBundle);
} else {
    curl_easy_setopt(curl, CURLOPT_CAINFO, "certs/cacert.pem");
}
```

Note: `CURL_CA_BUNDLE` is the standard environment variable that libcurl checks. Using it means the override follows the same convention that libcurl users expect. The fallback path `certs/cacert.pem` is relative to the working directory.

### 1.3: Copy `cacert.pem` to build output via CMake post-build step
The runtime working directory on Windows is `knottyyoga/server/knottyyoga_server/out/build/x64-Debug`, so the cert file needs to be copied there.

Add a CMake `add_custom_command(POST_BUILD ...)` that copies `certs/cacert.pem` from the source tree to `${CMAKE_BINARY_DIR}/certs/cacert.pem`. This ensures the file is available regardless of build configuration:

```cmake
add_custom_command(TARGET knottyyoga_the_server POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory "${CMAKE_BINARY_DIR}/certs"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_CURRENT_SOURCE_DIR}/certs/cacert.pem"
        "${CMAKE_BINARY_DIR}/certs/cacert.pem"
    COMMENT "Copying CA certificate bundle to build output"
)
```

On Linux (Docker), the same post-build step works — `CMAKE_BINARY_DIR` resolves to the Linux build directory.

### 1.4: Verification
- Run the C++ server on Windows via Visual Studio
- Navigate to `/shop/checkout/1`, enter test card `4111 1111 1111 1111`
- Payment should succeed (or get a valid Square API response, not an SSL error)
- Verify `certs/cacert.pem` exists in `out/build/x64-Debug/certs/` after building

---

# Section 2: Angular Dev Server with HTTPS

## Problem

`ng serve` currently serves over HTTP. Some browser features and third-party SDKs (like Square Web Payments) may require or work better with HTTPS, even in development.

## Solution: Self-Signed Certificate for `ng serve --ssl`

Angular CLI supports HTTPS via `ng serve --ssl`. For development, a self-signed certificate is sufficient.

## Implementation Steps

### 2.1: Generate a self-signed certificate
Use OpenSSL (available via Git Bash on Windows or WSL):

```bash
mkdir ui/certs
openssl req -x509 -newkey rsa:2048 -keyout ui/certs/localhost-key.pem -out ui/certs/localhost.pem -days 3650 -nodes -subj "/CN=localhost"
```

Add `ui/certs/` to `.gitignore` since each developer generates their own (or check it in — it's a self-signed cert with no security value).

### 2.2: Configure in `angular.json`
Add SSL options to the `serve.configurations.development` section:

```json
"development": {
  "proxyConfig": "src/proxy.conf.json",
  "buildTarget": "ui:build:development",
  "ssl": true,
  "sslCert": "certs/localhost.pem",
  "sslKey": "certs/localhost-key.pem"
}
```

Or just run with flags: `ng serve -c development --ssl --ssl-cert certs/localhost.pem --ssl-key certs/localhost-key.pem`

### 2.3: Update proxy configuration
The proxy in `proxy.conf.json` targets `http://127.0.0.1:18080` (the C++ server). This doesn't need to change — the proxy handles the HTTPS→HTTP bridge. Angular serves HTTPS to the browser, but proxies to the C++ server over HTTP.

### 2.4: Browser trust
On first visit to `https://localhost:4200`, the browser will show an untrusted certificate warning. Click through to accept, or install the cert in the Windows certificate store for a smoother experience. Alternatively, use `mkcert` (https://github.com/nicejongwook/mkcert) which creates locally-trusted certificates automatically.

### Open Questions
- [ ] Is HTTPS for `ng serve` actually needed right now? The Square Web Payments SDK appears to work over HTTP on localhost. This may only be needed if Square starts enforcing HTTPS for the SDK origin.
- [ ] Do you prefer generating certs per-developer (`.gitignore` them) or checking in a shared self-signed cert?

---

# Section 3: External HTTPS for Square Webhooks (ngrok)

## Problem

Square webhooks (payment notifications, refund callbacks, etc.) need to call endpoints on your server from the internet. During development, your C++ server runs on `localhost:18080` which is not publicly accessible. Square requires HTTPS webhook URLs.

## Solution: ngrok Tunnel

ngrok creates a public HTTPS URL that tunnels traffic to your local server. Square webhook events get forwarded through ngrok to your local C++ server.

## Implementation Steps

### 3.1: Install ngrok
- Download from https://ngrok.com/download or install via `choco install ngrok` on Windows
- Sign up for a free account at https://ngrok.com and get an auth token
- Configure: `ngrok config add-authtoken <your-token>`

### 3.2: Start the tunnel
```bash
ngrok http 18080
```

This produces output like:
```
Forwarding  https://abc123.ngrok-free.app -> http://localhost:18080
```

### 3.3: Configure Square webhook URL
In the Square Developer Dashboard (https://developer.squareup.com/apps):
1. Select your Sandbox app
2. Go to Webhooks
3. Add subscription with the ngrok URL: `https://abc123.ngrok-free.app/api/webhook`
4. Select events to subscribe to (e.g., `payment.completed`)

### 3.4: Add webhook endpoint to C++ server
This is a future implementation task. When needed:
- Create `endpoints/webhook_endpoint.cpp`
- Verify Square webhook signatures (using the Square signature key)
- Handle the webhook payload

### 3.5: CORS considerations
The Crow server's CORS handler currently allows `http://localhost:4200`. When using ngrok, the requests come from Square's servers (not a browser), so CORS is not an issue for webhooks. However, if you need the Angular app to be accessed via ngrok too, you'd need to update CORS.

### Workflow for Development with Webhooks
1. Start PostgreSQL: `cd database_server && load_container.cmd`
2. Start C++ server (Visual Studio or command line, port 18080)
3. Start ngrok: `ngrok http 18080`
4. Copy ngrok URL to Square Developer Dashboard webhook config
5. Start Angular: `ng serve -c development`
6. Test payment flow — Square webhook fires → ngrok → localhost:18080

### Open Questions
- [ ] The free ngrok tier gives a random URL each restart. Do you want to pay for a fixed subdomain, or is re-configuring the Square Dashboard each time acceptable?
- [ ] Are there specific Square webhook events you need now, or is this entirely future work?
- [ ] Does the Crow server need any changes to accept requests from ngrok (e.g., checking `X-Forwarded-For` headers)?

---

# Section 4: Production HTTPS on AWS (Future)

## Problem

Eventually the C++ web server will be hosted on AWS, serving both the Angular app (static files) and the APIs. It needs to handle HTTPS for real users.

## High-Level Architecture

```
                    ┌──────────────────┐
    Users ──HTTPS──▶│  AWS ALB / NLB   │──HTTP──▶ C++ Crow Server (port 8080)
                    │  (SSL termination)│         (serves /api/* endpoints)
                    │  ACM Certificate  │         (serves Angular static files)
                    └──────────────────┘
```

### Option A: SSL Termination at Load Balancer (Recommended)
- AWS Application Load Balancer (ALB) handles HTTPS
- SSL certificate from AWS Certificate Manager (ACM) — free, auto-renewing
- ALB forwards HTTP to the C++ server on port 8080
- Crow server stays HTTP-only (no code changes needed for HTTPS)
- ALB handles health checks, can scale to multiple instances later

### Option B: SSL Termination at Crow (Direct HTTPS)
- Crow supports SSL via `app.ssl_file("cert.pem", "key.pem")`
- Would need Let's Encrypt certificates with auto-renewal (certbot)
- More complex to manage, no load balancing
- Only makes sense if running on a single VM without a load balancer

### Option C: Reverse Proxy (nginx/Caddy)
- nginx or Caddy sits in front of Crow
- Handles SSL termination, static file serving, caching
- Crow only handles `/api/*` requests
- Good middle ground if not using AWS ALB

## Key Decisions (Not Needed Now)

| Decision | Options | Notes |
|----------|---------|-------|
| SSL termination | ALB vs nginx vs Crow direct | ALB recommended for AWS |
| Certificate | ACM (ALB only) vs Let's Encrypt | ACM is free and auto-renews |
| Static files | Crow serves them vs S3+CloudFront vs nginx | Crow serving is simplest for thin slice |
| Deployment | EC2 vs ECS vs Lambda | EC2 simplest, ECS for containers |
| Domain | Custom domain needed | Route 53 or external DNS |

## What Crow Needs to Serve Angular

When the C++ server serves the Angular app in production:
1. Build Angular: `ng build` produces files in `ui/dist/`
2. Crow serves these as static files for any non-`/api/` route
3. All routes that don't match `/api/*` return `index.html` (Angular handles client-side routing)
4. This is a future implementation task

## Open Questions
- [ ] What AWS services are you already using or planning to use?
- [ ] Do you have a domain name in mind?
- [ ] Is EC2 the planned hosting, or are you considering containers (ECS/EKS)?
- [ ] Timeline — when do you need production HTTPS?

---

# Priority Order

| Section | Priority | Blocking? | Effort |
|---------|----------|-----------|--------|
| 1. Fix SSL for libcurl (Square API) | **HIGH — Do Now** | Blocks payment testing | Small (1-2 hours) |
| 2. Angular HTTPS dev server | LOW | Not blocking anything | Small (30 min) |
| 3. ngrok for webhooks | MEDIUM | Blocks webhook development | Small (setup only) |
| 4. Production HTTPS on AWS | LOW | Future work | Large |