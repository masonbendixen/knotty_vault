---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/2/2026
Version: 0.1
tags: 
---
# Overview

In the planning directory is Payment Design Document.md. Using that document and the code base as well as what you can find about setting up Square for credit card payment, let's work on getting things setup so I can use the Web Payments SDK from Angular on the client and the Payment SDK from C++ on the server. I want to put together an exact plan for what I need to so and a step by step guide for how to do it here. Please go into plan mode and leave this overview section in this document alone but place the plan in sections after Overview.

# Summary

Set up Square payment integration for Knotty Yoga with:
- **Client**: Angular Web Payments SDK for card tokenization
- **Server**: C++ libcurl-based HTTP client for Square Payments API

---

# Phase 1: Square Developer Account Setup

## 1.1 Create Square Developer Account

1. Go to [Square Developer Dashboard](https://developer.squareup.com/apps)
2. Sign in with existing Square account or create new one
3. Click "+" to create a new application
4. Name it "Knotty Yoga" (or similar)

## 1.2 Get Sandbox Credentials

From the Developer Console:
1. Select your application
2. Set mode to **Sandbox** (toggle at top of page)
3. Navigate to **Credentials** in left pane
4. Copy these values to a secure location:
   - **Sandbox Application ID** (starts with `sandbox-sq0idb-`)
   - **Sandbox Access Token** (starts with `EAAAl...`)
5. Navigate to **Locations** in left pane
6. Copy the **Sandbox Location ID**

**Important**: These are secrets - never commit to version control.

## 1.3 Credentials Summary

| Credential | Where Used | Example Format |
|------------|------------|----------------|
| Application ID | Angular (client) | `sandbox-sq0idb-XXXX` |
| Location ID | Angular (client) | `LXXXX` |
| Access Token | C++ server | `EAAAl...` (long token) |

---

# Phase 2: Angular Client Setup

## 2.1 Add Square SDK Script

**File**: `ui/src/index.html`

Add to `<head>`:
```html
<!-- Square Web Payments SDK (Sandbox) -->
<script src="https://sandbox.web.squarecdn.com/v1/square.js"></script>
```

For production, remove "sandbox":
```html
<script src="https://web.squarecdn.com/v1/square.js"></script>
```

## 2.2 Add Environment Configuration

**File**: `ui/src/environments/environment.ts` (production)
```typescript
export const environment = {
  production: true,
  square: {
    applicationId: 'sq0idp-XXXX',  // Production App ID
    locationId: 'LXXXX'            // Production Location ID
  }
};
```

**File**: `ui/src/environments/environment.development.ts` (sandbox)
```typescript
export const environment = {
  production: false,
  square: {
    applicationId: 'sandbox-sq0idb-XXXX',  // Sandbox App ID
    locationId: 'LXXXX'                     // Sandbox Location ID
  }
};
```

## 2.3 Create TypeScript Declaration

**File**: `ui/src/types/square.d.ts`
```typescript
declare global {
  interface Window {
    Square: {
      payments(appId: string, locationId: string): Promise<Payments>;
    };
  }
}

interface Payments {
  card(): Promise<Card>;
}

interface Card {
  attach(selector: string): Promise<void>;
  tokenize(): Promise<TokenizeResult>;
}

interface TokenizeResult {
  status: 'OK' | 'ERROR';
  token?: string;
  errors?: Array<{ message: string }>;
}

export {};
```

## 2.4 Create Square Payment Service

**File**: `ui/src/app/portal/services/square-payment.service.ts`
```typescript
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class SquarePaymentService {
  private payments: Payments | null = null;
  private card: Card | null = null;

  async initialize(): Promise<void> {
    if (!window.Square) {
      throw new Error('Square SDK not loaded');
    }
    this.payments = await window.Square.payments(
      environment.square.applicationId,
      environment.square.locationId
    );
  }

  async attachCard(containerId: string): Promise<void> {
    if (!this.payments) await this.initialize();
    this.card = await this.payments!.card();
    await this.card.attach(containerId);
  }

  async tokenizeCard(): Promise<string> {
    if (!this.card) throw new Error('Card not attached');
    const result = await this.card.tokenize();
    if (result.status !== 'OK' || !result.token) {
      throw new Error(result.errors?.[0]?.message || 'Tokenization failed');
    }
    return result.token;
  }
}
```

## 2.5 Update Content Security Policy (if applicable)

If your app has CSP headers, add Square domains:
```
script-src 'self' https://sandbox.web.squarecdn.com https://web.squarecdn.com;
frame-src 'self' https://sandbox.web.squarecdn.com https://web.squarecdn.com;
connect-src 'self' https://pci-connect.squareup.com;
```

---

# Phase 3: C++ Server Setup

## 3.1 Add Secret Keys

**File**: `server/knottyyoga_server/src/secrets/secret_keys.h`

Add new constants:
```cpp
// Square API credentials
inline constexpr std::string_view kSquareAccessToken = "square_access_token";
inline constexpr std::string_view kSquareEnvironment = "square_environment";  // "sandbox" or "production"
```

## 3.2 Create HTTP Client Wrapper

**File**: `server/knottyyoga_server/src/util/http/http_client.h`
```cpp
#pragma once

#include <string>
#include <map>

namespace HttpClient {

struct HttpResponse {
    int statusCode;
    std::string body;
    std::map<std::string, std::string> headers;
};

struct HttpRequest {
    std::string url;
    std::string method;  // "GET", "POST", etc.
    std::string body;
    std::map<std::string, std::string> headers;
};

HttpResponse Execute(const HttpRequest& request);

}  // namespace HttpClient
```

**File**: `server/knottyyoga_server/src/util/http/http_client.cpp`
```cpp
#include "http_client.h"
#include <curl/curl.h>
#include <stdexcept>

namespace HttpClient {

namespace {
size_t WriteCallback(void* contents, size_t size, size_t nmemb, std::string* output) {
    size_t totalSize = size * nmemb;
    output->append(static_cast<char*>(contents), totalSize);
    return totalSize;
}
}  // namespace

HttpResponse Execute(const HttpRequest& request) {
    CURL* curl = curl_easy_init();
    if (!curl) {
        throw std::runtime_error("Failed to initialize CURL");
    }

    HttpResponse response;
    std::string responseBody;

    curl_easy_setopt(curl, CURLOPT_URL, request.url.c_str());
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &responseBody);

    // Set method and body
    if (request.method == "POST") {
        curl_easy_setopt(curl, CURLOPT_POST, 1L);
        curl_easy_setopt(curl, CURLOPT_POSTFIELDS, request.body.c_str());
    }

    // Set headers
    struct curl_slist* headers = nullptr;
    for (const auto& [key, value] : request.headers) {
        std::string header = key + ": " + value;
        headers = curl_slist_append(headers, header.c_str());
    }
    curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

    // Execute
    CURLcode res = curl_easy_perform(curl);
    if (res != CURLE_OK) {
        curl_slist_free_all(headers);
        curl_easy_cleanup(curl);
        throw std::runtime_error(std::string("CURL error: ") + curl_easy_strerror(res));
    }

    long httpCode = 0;
    curl_easy_getinfo(curl, CURLINFO_RESPONSE_CODE, &httpCode);

    curl_slist_free_all(headers);
    curl_easy_cleanup(curl);

    response.statusCode = static_cast<int>(httpCode);
    response.body = responseBody;
    return response;
}

}  // namespace HttpClient
```

## 3.3 Create Square Client

**File**: `server/knottyyoga_server/src/square/square_client.h`
```cpp
#pragma once

#include <string>
#include <memory>
#include "util/json_value.h"

namespace Square {

struct PaymentResult {
    std::string paymentId;
    std::string status;
    int64_t amountCents;
    std::string rawJson;
};

class SquareClient {
public:
    SquareClient(std::string accessToken, bool sandbox);

    PaymentResult CreatePayment(
        const std::string& sourceId,
        int64_t amountCents,
        const std::string& currency,
        const std::string& idempotencyKey,
        const std::string& note = "");

private:
    std::string accessToken_;
    std::string baseUrl_;
    static constexpr std::string_view kApiVersion = "2026-01-22";

    Json::Value Post(const std::string& endpoint, const Json::Value& body);
};

using SquareClientPtr = std::shared_ptr<SquareClient>;

}  // namespace Square
```

**File**: `server/knottyyoga_server/src/square/square_client.cpp`
```cpp
#include "square_client.h"
#include "util/http/http_client.h"
#include <stdexcept>

namespace Square {

SquareClient::SquareClient(std::string accessToken, bool sandbox)
    : accessToken_(std::move(accessToken))
    , baseUrl_(sandbox
        ? "https://connect.squareupsandbox.com/v2"
        : "https://connect.squareup.com/v2") {
}

Json::Value SquareClient::Post(const std::string& endpoint, const Json::Value& body) {
    HttpClient::HttpRequest request;
    request.url = baseUrl_ + endpoint;
    request.method = "POST";
    request.body = body.ToString();
    request.headers = {
        {"Authorization", "Bearer " + accessToken_},
        {"Content-Type", "application/json"},
        {"Square-Version", std::string(kApiVersion)}
    };

    auto response = HttpClient::Execute(request);
    auto json = Json::Value::FromText(response.body);

    if (response.statusCode >= 400) {
        std::string errorMsg = "Square API error: " + std::to_string(response.statusCode);
        if (json.HasKey("errors")) {
            auto errors = json["errors"];
            if (errors.IsArray() && errors.ArraySize() > 0) {
                errorMsg += " - " + errors[0]["detail"].GetString();
            }
        }
        throw std::runtime_error(errorMsg);
    }

    return json;
}

PaymentResult SquareClient::CreatePayment(
    const std::string& sourceId,
    int64_t amountCents,
    const std::string& currency,
    const std::string& idempotencyKey,
    const std::string& note) {

    Json::Value body = Json::Value::Object({
        {"source_id", sourceId},
        {"idempotency_key", idempotencyKey},
        {"amount_money", Json::Value::Object({
            {"amount", amountCents},
            {"currency", currency}
        })}
    });

    if (!note.empty()) {
        body["note"] = note;
    }

    auto response = Post("/payments", body);
    auto payment = response["payment"];

    PaymentResult result;
    result.paymentId = payment["id"].GetString();
    result.status = payment["status"].GetString();
    result.amountCents = payment["amount_money"]["amount"].GetInt64();
    result.rawJson = response.ToString();
    return result;
}

}  // namespace Square
```

## 3.4 Add CMakeLists.txt for New Modules

**File**: `server/knottyyoga_server/src/util/http/CMakeLists.txt`
```cmake
target_sources(knotty_yoga_core PRIVATE
    http_client.cpp
)

target_link_libraries(knotty_yoga_core PUBLIC ${CURL_LIB})
```

**File**: `server/knottyyoga_server/src/square/CMakeLists.txt`
```cmake
target_sources(knotty_yoga_core PRIVATE
    square_client.cpp
)
```

Update `server/knottyyoga_server/src/CMakeLists.txt` to add:
```cmake
add_subdirectory(util/http)
add_subdirectory(square)
```

## 3.5 Store Credentials in Database

Using existing secrets infrastructure, add credentials via SQL or admin endpoint:

```sql
INSERT INTO config_secrets (name, value) VALUES
    ('square_access_token', 'EAAAl...');  -- Your sandbox token
INSERT INTO config_secrets (name, value) VALUES
    ('square_environment', 'sandbox');
```

Or use the SecretsHelper in code:
```cpp
secretsHelper->AddSecret(transaction, Secrets::kSquareAccessToken, "EAAAl...");
secretsHelper->AddSecret(transaction, Secrets::kSquareEnvironment, "sandbox");
```

---

# Phase 4: Integration & Testing

## 4.1 Test Card Numbers (Sandbox)

| Card Number | Result |
|-------------|--------|
| 4111 1111 1111 1111 | Success |
| 5105 1051 0510 5100 | Success (Mastercard) |
| 4000 0000 0000 0002 | Card declined |
| 4000 0000 0000 0010 | CVV failure |

Use any future expiration date and any 3-digit CVV.

## 4.2 End-to-End Test Flow

1. **Start Angular dev server**: `ng serve --configuration=development`
2. **Start C++ server** with sandbox credentials in database
3. **Open browser** to checkout page
4. **Enter test card** (4111 1111 1111 1111)
5. **Submit payment** → Angular tokenizes → Server creates payment
6. **Verify** in Square Developer Dashboard → Transactions

## 4.3 Webhook Testing (Optional for Phase 0)

For subscription webhooks later:
1. Install ngrok: `npm install -g ngrok` or download from ngrok.com
2. Run: `ngrok http 18080`
3. Copy HTTPS URL to Square Developer Dashboard → Webhooks
4. Subscribe to relevant events (payment.created, etc.)

---

# Phase 5: Production Checklist

When ready for production:

## 5.1 Angular Changes
- [ ] Update `environment.ts` with production Application ID and Location ID
- [ ] Change SDK script from `sandbox.web.squarecdn.com` to `web.squarecdn.com`

## 5.2 Server Changes
- [ ] Update `square_access_token` in database with production token
- [ ] Update `square_environment` to `"production"`

## 5.3 Square Dashboard
- [ ] Switch Developer Console to **Production** mode
- [ ] Verify production credentials are different from sandbox
- [ ] Set up production webhook URLs

## 5.4 Security
- [ ] Ensure access token is never logged
- [ ] Verify HTTPS is enforced
- [ ] Test error handling for declined cards

---

# Files to Create/Modify

## New Files

| File | Purpose |
|------|---------|
| `ui/src/types/square.d.ts` | TypeScript declarations for Square SDK |
| `ui/src/app/portal/services/square-payment.service.ts` | Angular service for card tokenization |
| `server/.../src/util/http/http_client.h` | HTTP client header |
| `server/.../src/util/http/http_client.cpp` | HTTP client implementation |
| `server/.../src/util/http/CMakeLists.txt` | CMake for HTTP module |
| `server/.../src/square/square_client.h` | Square client header |
| `server/.../src/square/square_client.cpp` | Square client implementation |
| `server/.../src/square/CMakeLists.txt` | CMake for Square module |

## Modified Files

| File | Change |
|------|--------|
| `ui/src/index.html` | Add Square SDK script |
| `ui/src/environments/environment.ts` | Add Square config |
| `ui/src/environments/environment.development.ts` | Add Square sandbox config |
| `server/.../src/secrets/secret_keys.h` | Add Square secret keys |
| `server/.../src/CMakeLists.txt` | Add subdirectories |

---

# Verification

1. **Build Angular**: `cd ui && npm install && ng build`
2. **Build C++ Server**: `cd server/knottyyoga_server/build && cmake .. && make`
3. **Insert test credentials** into database
4. **Test tokenization** in browser console:
   ```javascript
   const payments = await Square.payments('sandbox-sq0idb-...', 'L...');
   const card = await payments.card();
   await card.attach('#card-container');
   const result = await card.tokenize();
   console.log(result.token);
   ```
5. **Test server** with curl:
   ```bash
   curl -X POST https://connect.squareupsandbox.com/v2/payments \
     -H "Authorization: Bearer YOUR_SANDBOX_TOKEN" \
     -H "Content-Type: application/json" \
     -H "Square-Version: 2026-01-22" \
     -d '{"source_id":"TOKEN_FROM_STEP_4","idempotency_key":"unique-key","amount_money":{"amount":100,"currency":"USD"}}'
   ```

---

# Sources

- [Square Web Payments SDK Overview](https://developer.squareup.com/docs/web-payments/overview)
- [Add SDK to Web Client](https://developer.squareup.com/docs/web-payments/quickstart/add-sdk-to-web-client)
- [Square Payments API - CreatePayment](https://developer.squareup.com/reference/square/payments-api/create-payment)
- [Card Payments Guide](https://developer.squareup.com/docs/payments-api/take-payments/card-payments)
- [Using the REST API](https://developer.squareup.com/docs/build-basics/general-considerations/using-rest-api)

---

# Implementation Checklist

## Milestone 1: Square Developer Account Setup
- [x] Create Square Developer account at developer.squareup.com ✅ 2026-02-02
	- Username: knottyyoga@hotmail.com
	- Password: G#-5qd?:T$8m/zf
- [x] Create new application named "Knotty Yoga" ✅ 2026-02-02
	- Done
- [x] Switch to **Sandbox** mode in Developer Console ✅ 2026-02-02
- [x] Copy **Sandbox Application ID** (starts with `sandbox-sq0idb-`) ✅ 2026-02-02
	- sandbox-sq0idb-B1PoAtwzV7eEmN3u8FHLyQ
- [x] Copy **Sandbox Access Token** (starts with `EAAAl...`) ✅ 2026-02-02
	- EAAAl3eaBCzTnAZR_BJTKsYz0gbNTXz3KP4iOPwCdoBgB7tOaLlEMG6Dpqb6pvNO
- [x] Copy **Sandbox Location ID** (from Locations tab) ✅ 2026-02-02
	- NWLEQ37Z06H6JEC
- [x] Store credentials securely (NOT in version control) ✅ 2026-02-02

**Milestone Complete When**: All three sandbox credentials are saved securely

---

## Milestone 2: Angular Client Setup
- [x] Add Square SDK script to `ui/src/index.html` ✅ 2026-02-02
- [x] Add `square` config to `ui/src/environments/environment.ts` (production) ✅ 2026-02-02
- [x] Add `square` config to `ui/src/environments/environment.development.ts` (sandbox) ✅ 2026-02-02
- [x] Create `ui/src/types/square.d.ts` with TypeScript declarations ✅ 2026-02-02
- [x] Create `ui/src/app/portal/services/square-payment.service.ts` ✅ 2026-02-02
- [x] Build Angular: `cd ui && ng build` ✅ 2026-02-02
- [x] Verify no TypeScript errors ✅ 2026-02-02

**Milestone Complete When**: Angular builds successfully with Square service

---

## Milestone 3: C++ Server Setup
- [x] Add Square secret key constants to `server/.../src/secrets/secret_keys.h` ✅ 2026-02-03
- [x] Create `server/.../src/util/http/` directory ✅ 2026-02-03
- [x] Create `http_client.h` and `http_client.cpp` ✅ 2026-02-03
- [x] Create `server/.../src/util/http/CMakeLists.txt` ✅ 2026-02-03
- [x] Create `server/.../src/square/` directory ✅ 2026-02-03
- [x] Create `square_client.h` and `square_client.cpp` ✅ 2026-02-03
- [x] Create `server/.../src/square/CMakeLists.txt` ✅ 2026-02-03
- [x] Update `server/.../src/CMakeLists.txt` with new subdirectories ✅ 2026-02-03
- [x] Build server: `cmake .. && make` ✅ 2026-02-03
- [x] Verify no compilation errors ✅ 2026-02-03

**Milestone Complete When**: Server builds successfully with Square client

---

## Milestone 4: Database Credentials Setup
- [x] Connect to PostgreSQL database ✅ 2026-02-03
- [x] Insert `square_access_token` into `config_secrets` table ✅ 2026-02-03
- [x] Insert `square_environment` = `'sandbox'` into `config_secrets` table ✅ 2026-02-03
- [x] Verify secrets can be retrieved by server code ✅ 2026-02-03

**Milestone Complete When**: Server can read Square credentials from database

---

## Milestone 5: End-to-End Testing
- [ ] Start Angular dev server (`ng serve --configuration=development`)
- [ ] Start C++ server with sandbox credentials
- [ ] Create a test page/component with card container div
- [ ] Test Square SDK loads in browser console:
  ```javascript
  const payments = await Square.payments('sandbox-sq0idb-...', 'L...');
  console.log('Square initialized:', payments);
  ```
- [ ] Test card form attaches to container
- [ ] Test tokenization with test card (4111 1111 1111 1111)
- [ ] Verify token is returned
- [ ] Test server payment creation with curl (see Phase 4)
- [ ] Verify payment appears in Square Developer Dashboard → Transactions

**Milestone Complete When**: Can process test payment from browser through server

---

## Milestone 6: Production Deployment (When Ready)
- [ ] Get production credentials from Square Developer Console
- [ ] Update `ui/src/environments/environment.ts` with production IDs
- [ ] Change SDK script from `sandbox.web.squarecdn.com` to `web.squarecdn.com`
- [ ] Update `square_access_token` in database with production token
- [ ] Update `square_environment` to `'production'` in database
- [ ] Set up production webhook URLs in Square Dashboard
- [ ] Test with real card (small amount, then refund)
- [ ] Verify production payment in Square Dashboard

**Milestone Complete When**: Production payment processed successfully

---

## Quick Reference: Test Card Numbers

| Card | Number | Result |
|------|--------|--------|
| Visa | 4111 1111 1111 1111 | Success |
| Mastercard | 5105 1051 0510 5100 | Success |
| Declined | 4000 0000 0000 0002 | Card declined |
| CVV Fail | 4000 0000 0000 0010 | CVV failure |

*Use any future expiration date and any 3-digit CVV for sandbox testing.*

---

# Implementation Notes

## Phase 2: Angular Client Implementation Details

### Design Decision: Dynamic Script Loading

Rather than using separate `index.html` files or conditional logic scattered through the code, we'll use **dynamic script loading** based on the environment configuration. This approach:

- Keeps all Square configuration together in environment files
- Follows the existing file replacement pattern in `angular.json`
- Avoids duplicating `index.html`
- Requires no changes to `angular.json` build configuration

### Files to Create

#### 1. `ui/src/environments/environment.ts` (modify existing)

Add Square configuration for **production**:

```typescript
export const environment = {
  production: true,
  // ... existing config ...
  square: {
    applicationId: 'sq0idp-XXXX',           // Production App ID (get when ready)
    locationId: 'LXXXX',                     // Production Location ID
    scriptUrl: 'https://web.squarecdn.com/v1/square.js'
  }
};
```

#### 2. `ui/src/environments/environment.development.ts` (modify existing)

Add Square configuration for **sandbox**:

```typescript
export const environment = {
  production: false,
  // ... existing config ...
  square: {
    applicationId: 'sandbox-sq0idb-B1PoAtwzV7eEmN3u8FHLyQ',
    locationId: 'LXXXX',                     // Sandbox Location ID
    scriptUrl: 'https://sandbox.web.squarecdn.com/v1/square.js'
  }
};
```

#### 3. `ui/src/types/square.d.ts` (new file)

TypeScript declarations for the Square SDK global:

```typescript
declare global {
  interface Window {
    Square: {
      payments(appId: string, locationId: string): Promise<Payments>;
    };
  }
}

interface Payments {
  card(): Promise<Card>;
}

interface Card {
  attach(selector: string): Promise<void>;
  tokenize(): Promise<TokenizeResult>;
}

interface TokenizeResult {
  status: 'OK' | 'ERROR';
  token?: string;
  errors?: Array<{ message: string }>;
}

export {};
```

#### 4. `ui/src/app/portal/services/square-payment.service.ts` (new file)

Angular service with dynamic script loading:

```typescript
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class SquarePaymentService {
  private payments: Payments | null = null;
  private card: Card | null = null;
  private scriptLoaded = false;

  /**
   * Dynamically loads the Square SDK script based on environment config.
   * Only loads once; subsequent calls return immediately.
   */
  private loadScript(): Promise<void> {
    return new Promise((resolve, reject) => {
      if (this.scriptLoaded || window.Square) {
        this.scriptLoaded = true;
        resolve();
        return;
      }

      const script = document.createElement('script');
      script.src = environment.square.scriptUrl;
      script.onload = () => {
        this.scriptLoaded = true;
        resolve();
      };
      script.onerror = () => reject(new Error('Failed to load Square SDK'));
      document.head.appendChild(script);
    });
  }

  /**
   * Initializes the Square Payments SDK.
   * Automatically loads the script if not already loaded.
   */
  async initialize(): Promise<void> {
    await this.loadScript();

    if (!window.Square) {
      throw new Error('Square SDK not available after loading');
    }

    this.payments = await window.Square.payments(
      environment.square.applicationId,
      environment.square.locationId
    );
  }

  /**
   * Attaches the Square card input form to a DOM element.
   * @param containerId CSS selector for the container element (e.g., '#card-container')
   */
  async attachCard(containerId: string): Promise<void> {
    if (!this.payments) {
      await this.initialize();
    }
    this.card = await this.payments!.card();
    await this.card.attach(containerId);
  }

  /**
   * Tokenizes the card data entered by the user.
   * @returns The payment token to send to the server
   * @throws Error if tokenization fails
   */
  async tokenizeCard(): Promise<string> {
    if (!this.card) {
      throw new Error('Card not attached. Call attachCard() first.');
    }

    const result = await this.card.tokenize();

    if (result.status !== 'OK' || !result.token) {
      const errorMessage = result.errors?.[0]?.message || 'Card tokenization failed';
      throw new Error(errorMessage);
    }

    return result.token;
  }

  /**
   * Destroys the card form and resets state.
   * Call this when navigating away from the payment page.
   */
  destroy(): void {
    this.card = null;
    this.payments = null;
    // Note: We don't unload the script as it may be needed again
  }
}
```

### Implementation Steps

1. **Read existing environment files** to understand current structure
2. **Add `square` config object** to both environment files
3. **Create TypeScript declarations** file for Square SDK types
4. **Create SquarePaymentService** with dynamic script loading
5. **Verify build** compiles without errors
6. **Test in browser** that SDK loads and initializes correctly

### No Changes Needed

- `ui/src/index.html` - No script tag needed (loaded dynamically)
- `ui/angular.json` - No additional file replacements needed
- CSP headers - Only needed if you have Content Security Policy configured

### Usage Example

In a component:

```typescript
import { SquarePaymentService } from '@portal/services/square-payment.service';

@Component({
  template: `<div id="card-container"></div><button (click)="pay()">Pay</button>`
})
export class CheckoutComponent implements AfterViewInit {
  constructor(private squarePayment: SquarePaymentService) {}

  async ngAfterViewInit() {
    await this.squarePayment.attachCard('#card-container');
  }

  async pay() {
    try {
      const token = await this.squarePayment.tokenizeCard();
      // Send token to your server endpoint
      console.log('Payment token:', token);
    } catch (error) {
      console.error('Payment failed:', error);
    }
  }
}
```

---

## Phase 3: C++ Server Implementation Details

### Design Decision: Testable HTTP Client Interface

The codebase follows a consistent pattern for external dependencies that need to be mocked in tests:

| Component | Interface | Test Implementation | Factory |
|-----------|-----------|---------------------|---------|
| `SecretsHelper` | Abstract class with virtual methods | `Test::SecretsHelperTest` (in-memory) | `MakeSecretsHelper()` / `MakeTestSecretsHelper()` |
| `MailHelper` | Abstract class with `SendMail()` | `Test::TestMailHelper` (captures messages) | `MakeMailHelper()` / `MakeTestMailHelper()` |
| `CookieManager` | Abstract class with `SetCookie()`/`GetCookies()` | `Test::CookieManagerTest` (in-memory) | `MakeCookieManager()` / `MakeCookieManagerTest()` |
| `TransactionProvider` | Abstract class with `RunInTransaction()` | Test version that rolls back | Factory functions |

**Pattern elements:**
1. Abstract interface class with pure virtual methods
2. `using XxxPtr = std::shared_ptr<Xxx>` typedef
3. Production factory function `MakeXxx()`
4. Test implementation in `xxx_test_util.h` under `Test::` namespace
5. Test factory function `MakeTestXxx()`
6. Test implementations often expose additional methods for verification (e.g., `TestMailHelper::GetMessages()`)

### Proposed Architecture

Apply this pattern to HTTP and Square clients:

```
┌─────────────────────────────────────────────────────────────────┐
│                         SquareClient                            │
│  - Takes HttpClientPtr via constructor (dependency injection)   │
│  - CreatePayment(), GetPayment(), etc.                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HttpClient (interface)                       │
│  virtual HttpResponse Execute(const HttpRequest&) = 0          │
└─────────────────────────────────────────────────────────────────┘
              │                                    │
              ▼                                    ▼
┌─────────────────────────┐         ┌─────────────────────────────┐
│   HttpClientImpl        │         │   Test::TestHttpClient      │
│   (uses libcurl)        │         │   - Stores requests         │
│                         │         │   - Returns fake responses  │
│   MakeHttpClient()      │         │   - GetRequests() for verify│
└─────────────────────────┘         │   MakeTestHttpClient()      │
                                    └─────────────────────────────┘
```

### HttpClient Interface

**File**: `server/knottyyoga_server/src/util/http/http_client.h`

```cpp
#pragma once

#include <string>
#include <map>
#include <memory>

namespace Http {

struct HttpResponse {
    int statusCode;
    std::string body;
    std::map<std::string, std::string> headers;
};

struct HttpRequest {
    std::string url;
    std::string method;  // "GET", "POST", etc.
    std::string body;
    std::map<std::string, std::string> headers;
};

class HttpClient {
public:
    virtual ~HttpClient() = default;

    virtual HttpResponse Execute(const HttpRequest& request) = 0;

protected:
    HttpClient() = default;
    HttpClient(const HttpClient&) = default;
    HttpClient& operator=(const HttpClient&) = default;
};

using HttpClientPtr = std::shared_ptr<HttpClient>;

// Factory for production implementation (uses libcurl)
HttpClientPtr MakeHttpClient();

}  // namespace Http
```

### Test HttpClient

**File**: `server/knottyyoga_server/src/util/http/http_client_test_util.h`

```cpp
#pragma once

#include "http_client.h"
#include <vector>
#include <functional>

namespace Http {
namespace Test {

using RequestList = std::vector<HttpRequest>;

class TestHttpClient : public HttpClient {
public:
    TestHttpClient() = default;
    TestHttpClient(const TestHttpClient&) = delete;
    TestHttpClient& operator=(const TestHttpClient&) = delete;
    ~TestHttpClient() override = default;

    HttpResponse Execute(const HttpRequest& request) override;

    // Test helpers
    const RequestList& GetRequests() const { return requests_; }
    void ClearRequests() { requests_.clear(); }

    // Set up fake responses
    using ResponseHandler = std::function<HttpResponse(const HttpRequest&)>;
    void SetResponseHandler(ResponseHandler handler) { responseHandler_ = handler; }

    // Simple helpers for common cases
    void SetNextResponse(HttpResponse response);
    void SetNextResponse(int statusCode, const std::string& body);

private:
    RequestList requests_;
    ResponseHandler responseHandler_;
    std::vector<HttpResponse> queuedResponses_;
};

using TestHttpClientPtr = std::shared_ptr<TestHttpClient>;

TestHttpClientPtr MakeTestHttpClient();

}  // namespace Test
}  // namespace Http
```

### SquareClient with Dependency Injection

**File**: `server/knottyyoga_server/src/square/square_client.h`

```cpp
#pragma once

#include <string>
#include <memory>
#include "util/http/http_client.h"
#include "util/json_value.h"

namespace Square {

struct PaymentResult {
    std::string paymentId;
    std::string status;
    int64_t amountCents;
    std::string rawJson;
};

class SquareClient {
public:
    // Constructor takes HttpClient for dependency injection
    SquareClient(Http::HttpClientPtr httpClient, std::string accessToken, bool sandbox);

    PaymentResult CreatePayment(
        const std::string& sourceId,
        int64_t amountCents,
        const std::string& currency,
        const std::string& idempotencyKey,
        const std::string& note = "");

private:
    Http::HttpClientPtr httpClient_;
    std::string accessToken_;
    std::string baseUrl_;
    static constexpr std::string_view kApiVersion = "2026-01-22";

    Json::Value Post(const std::string& endpoint, const Json::Value& body);
};

using SquareClientPtr = std::shared_ptr<SquareClient>;

// Factory for production use
SquareClientPtr MakeSquareClient(std::string accessToken, bool sandbox);

}  // namespace Square
```

### Test Example

```cpp
#include "square/square_client.h"
#include "util/http/http_client_test_util.h"
#include <gtest/gtest.h>

TEST(SquareClientTest, CreatePaymentSendsCorrectRequest) {
    // Arrange: Set up test HTTP client with fake response
    auto testHttpClient = Http::Test::MakeTestHttpClient();
    testHttpClient->SetNextResponse(200, R"({
        "payment": {
            "id": "pay_123",
            "status": "COMPLETED",
            "amount_money": {"amount": 1000, "currency": "USD"}
        }
    })");

    auto squareClient = std::make_shared<Square::SquareClient>(
        testHttpClient, "test-token", /*sandbox=*/true);

    // Act
    auto result = squareClient->CreatePayment(
        "cnon:card-nonce", 1000, "USD", "idempotency-123", "Test payment");

    // Assert: Verify the request was correct
    ASSERT_EQ(testHttpClient->GetRequests().size(), 1);
    auto& request = testHttpClient->GetRequests()[0];

    EXPECT_EQ(request.url, "https://connect.squareupsandbox.com/v2/payments");
    EXPECT_EQ(request.method, "POST");
    EXPECT_EQ(request.headers.at("Authorization"), "Bearer test-token");
    EXPECT_EQ(request.headers.at("Square-Version"), "2026-01-22");

    // Verify request body contains expected fields
    auto body = Json::Value::FromText(request.body);
    EXPECT_EQ(body["source_id"].GetString(), "cnon:card-nonce");
    EXPECT_EQ(body["amount_money"]["amount"].GetInt64(), 1000);
    EXPECT_EQ(body["idempotency_key"].GetString(), "idempotency-123");

    // Verify response was parsed correctly
    EXPECT_EQ(result.paymentId, "pay_123");
    EXPECT_EQ(result.status, "COMPLETED");
    EXPECT_EQ(result.amountCents, 1000);
}

TEST(SquareClientTest, CreatePaymentHandlesApiError) {
    auto testHttpClient = Http::Test::MakeTestHttpClient();
    testHttpClient->SetNextResponse(400, R"({
        "errors": [{"detail": "Invalid source_id"}]
    })");

    auto squareClient = std::make_shared<Square::SquareClient>(
        testHttpClient, "test-token", /*sandbox=*/true);

    // Act & Assert
    EXPECT_THROW(
        squareClient->CreatePayment("invalid", 1000, "USD", "key"),
        std::runtime_error);
}
```

### Benefits of This Approach

1. **No network calls in tests**: Tests run fast and don't depend on external services
2. **Request verification**: Tests can verify exact headers, URLs, and body content sent to Square
3. **Error scenario testing**: Easy to simulate API errors, timeouts, malformed responses
4. **Consistent with codebase**: Follows the same pattern as MailHelper, SecretsHelper, etc.
5. **Production code unchanged**: The SquareClient logic is the same; only the HTTP transport is swapped

### Files to Create for Phase 3

| File | Purpose |
|------|---------|
| `src/util/http/http_client.h` | HttpClient interface |
| `src/util/http/http_client.cpp` | Production implementation (libcurl) |
| `src/util/http/http_client_test_util.h` | Test implementation header |
| `src/util/http/http_client_test_util.cpp` | Test implementation |
| `src/util/http/CMakeLists.txt` | CMake for HTTP module |
| `src/square/square_client.h` | SquareClient with DI |
| `src/square/square_client.cpp` | SquareClient implementation |
| `src/square/CMakeLists.txt` | CMake for Square module |
| `src/secrets/secret_keys.h` | Add Square secret key constants |
| `test/src/square/square_client_test.cpp` | Unit tests for SquareClient |

### Design Decisions (Resolved)

#### 1. SquareClient Interface

**Decision**: Yes, put SquareClient behind an interface with a test implementation.

**Rationale**: When testing endpoint handlers that use SquareClient, mocking at the HTTP level requires setting up JSON request/response fixtures for every test. A `TestSquareClient` can directly return `PaymentResult` objects, making tests cleaner and more focused.

**Updated design:**

```cpp
// square_client.h
namespace Square {

class SquareClient {
public:
    virtual ~SquareClient() = default;

    virtual PaymentResult CreatePayment(
        const std::string& sourceId,
        int64_t amountCents,
        const std::string& currency,
        const std::string& idempotencyKey,
        const std::string& note = "") = 0;

    // Future methods as needed:
    // virtual PaymentResult GetPayment(const std::string& paymentId) = 0;
    // virtual void RefundPayment(...) = 0;

protected:
    SquareClient() = default;
};

using SquareClientPtr = std::shared_ptr<SquareClient>;

// Production factory
SquareClientPtr MakeSquareClient(
    Http::HttpClientPtr httpClient,
    std::string accessToken,
    bool sandbox);

}  // namespace Square
```

```cpp
// square_client_test_util.h
namespace Square {
namespace Test {

class TestSquareClient : public SquareClient {
public:
    PaymentResult CreatePayment(...) override;

    // Test helpers
    void SetNextPaymentResult(PaymentResult result);
    void SetNextError(std::exception_ptr error);
    const std::vector<CreatePaymentArgs>& GetCreatePaymentCalls() const;

private:
    std::vector<CreatePaymentArgs> createPaymentCalls_;
    std::queue<std::variant<PaymentResult, std::exception_ptr>> responses_;
};

using TestSquareClientPtr = std::shared_ptr<TestSquareClient>;
TestSquareClientPtr MakeTestSquareClient();

}  // namespace Test
}  // namespace Square
```

---

#### 2. Exception Types and Error Mapping

**Decision**: Use specific exception types that can be caught and mapped to `ErrorResponse` functions.

**Rationale**: The existing `ErrorResponse` namespace (in `util/error_response.h`) already has `PaymentDeclined()` and `PaymentFailed()` that accept provider and provider code. Square exceptions should carry enough information to populate these.

**Square error categories** (from Square API):
- `AUTHENTICATION_ERROR` - Invalid/expired token
- `INVALID_REQUEST_ERROR` - Bad request data
- `RATE_LIMIT_ERROR` - Too many requests (429)
- `PAYMENT_METHOD_ERROR` - Card declined, CVV failure, etc.
- `REFUND_ERROR` - Refund-specific issues
- `API_ERROR` - Square server error (5xx)

**Proposed exception hierarchy:**

```cpp
// square/square_errors.h
namespace Square {

// Base exception for all Square errors
class SquareException : public std::exception {
public:
    SquareException(std::string category, std::string code, std::string detail);

    const char* what() const noexcept override { return detail_.c_str(); }
    const std::string& GetCategory() const { return category_; }
    const std::string& GetCode() const { return code_; }
    const std::string& GetDetail() const { return detail_; }

private:
    std::string category_;
    std::string code_;
    std::string detail_;
};

// Payment was declined (card issue, insufficient funds, etc.)
// Maps to ErrorResponse::PaymentDeclined()
class PaymentDeclinedException : public SquareException {
public:
    using SquareException::SquareException;
};

// Request was invalid (bad parameters, missing fields)
// Maps to ErrorResponse::ValidationError() or BadRequest()
class InvalidRequestException : public SquareException {
public:
    InvalidRequestException(std::string code, std::string detail, std::string field = "");
    const std::string& GetField() const { return field_; }
private:
    std::string field_;
};

// Rate limited - caller should retry with backoff
// Maps to ErrorResponse with 429 status
class RateLimitException : public SquareException {
public:
    using SquareException::SquareException;
};

// Square server error (5xx) - may be transient
// Maps to ErrorResponse::InternalError() or PaymentFailed()
class SquareServerException : public SquareException {
public:
    using SquareException::SquareException;
};

// Network/connection error - may be transient
class NetworkException : public std::exception {
public:
    explicit NetworkException(std::string message);
    const char* what() const noexcept override { return message_.c_str(); }
private:
    std::string message_;
};

}  // namespace Square
```

**Mapping to ErrorResponse in endpoint handlers:**

```cpp
crow::response HandlePayment(/* ... */) {
    try {
        auto result = squareClient->CreatePayment(/* ... */);
        return crow::response(200, result.ToJson());
    }
    catch (const Square::PaymentDeclinedException& e) {
        return ErrorResponse::PaymentDeclined(e.GetDetail(), "square", e.GetCode());
    }
    catch (const Square::InvalidRequestException& e) {
        return ErrorResponse::ValidationError(e.GetDetail(), e.GetField());
    }
    catch (const Square::RateLimitException& e) {
        // Return 503 to client, they can retry
        return ErrorResponse::Create(
            "payment-service-unavailable",
            "Payment Service Temporarily Unavailable",
            503,
            "The payment service is temporarily unavailable. Please try again.");
    }
    catch (const Square::SquareServerException& e) {
        return ErrorResponse::PaymentFailed(e.GetDetail(), "square", e.GetCode());
    }
    catch (const Square::NetworkException& e) {
        return ErrorResponse::InternalError("Unable to connect to payment service");
    }
}
```

---

#### 3. Retry Logic

**Decision**: Implement retry logic in `SquareClient`, not `HttpClient`, with caller control.

**Square's documented guidance** ([developer.squareup.com](https://developer.squareup.com/docs/build-basics/handling-errors)):

| Status | Retryable? | Action |
|--------|-----------|--------|
| 4xx (except 429) | No | Don't retry; fix the request |
| 429 Rate Limit | Yes | Retry with exponential backoff + jitter |
| 5xx Server Error | Yes | Retry with exponential backoff |
| Network timeout | Yes | Retry with exponential backoff |

**Square's requirements:**
- Apps in the App Marketplace **must** implement exponential backoff for 429 errors
- Use idempotency keys for all mutating operations (CreatePayment, etc.) to make retries safe
- Add random jitter to prevent thundering herd

**Proposed implementation:**

```cpp
// square_client.h
struct RetryPolicy {
    int maxRetries = 3;
    std::chrono::milliseconds initialDelay{500};
    std::chrono::milliseconds maxDelay{30000};
    double backoffMultiplier = 2.0;
    bool addJitter = true;

    static RetryPolicy Default() { return {}; }
    static RetryPolicy NoRetry() { return {0, {}, {}, 1.0, false}; }
};

class SquareClient {
public:
    // ... existing methods ...

    // Allow caller to override retry policy per-call
    virtual PaymentResult CreatePayment(
        const std::string& sourceId,
        int64_t amountCents,
        const std::string& currency,
        const std::string& idempotencyKey,
        const std::string& note = "",
        const RetryPolicy& retryPolicy = RetryPolicy::Default()) = 0;
};
```

**Why in SquareClient, not HttpClient:**
- SquareClient understands which errors are retryable (429, 5xx)
- SquareClient knows about idempotency keys (safe to retry)
- HttpClient stays simple and reusable for non-Square APIs
- Caller can disable retries for time-sensitive operations

**Idempotency keys** ensure CreatePayment retries are safe:
- Same key = Square returns original result, no duplicate charge
- Keys should be UUIDs (use Boost.Uuid or similar)
- Square recommends language-standard UUID generators

---

### Updated Files List

| File | Purpose |
|------|---------|
| `src/util/http/http_client.h` | HttpClient interface |
| `src/util/http/http_client.cpp` | Production implementation (libcurl) |
| `src/util/http/http_client_test_util.h` | Test HttpClient header |
| `src/util/http/http_client_test_util.cpp` | Test HttpClient implementation |
| `src/util/http/CMakeLists.txt` | CMake for HTTP module |
| `src/square/square_client.h` | SquareClient interface |
| `src/square/square_client.cpp` | Production SquareClient (with retry logic) |
| `src/square/square_client_test_util.h` | Test SquareClient header |
| `src/square/square_client_test_util.cpp` | Test SquareClient implementation |
| `src/square/square_errors.h` | Exception types |
| `src/square/CMakeLists.txt` | CMake for Square module |
| `src/secrets/secret_keys.h` | Add Square secret key constants |
| `test/src/square/square_client_test.cpp` | Unit tests for SquareClient |
| `test/src/util/http/http_client_test.cpp` | Unit tests for HttpClient |