# inventaory-organizer
A mobile-to-cloud app that tracks what's in your fridge and pushes you a recipe for ingredients about to expire. 
[fridge-app-architecture.md](https://github.com/user-attachments/files/29796934/fridge-app-architecture.md)

# Fridge Inventory App — Free-Forever Serverless Architecture

A mobile-to-cloud app that tracks what's in your fridge and pushes you a recipe
for ingredients about to expire. This build stays at **$0/month indefinitely** by
using only AWS *always-free* services plus free tiers from Firebase and Google.

The single most important design decision: **there is no API Gateway.** API Gateway
lost its always-free tier for AWS accounts created after 15 July 2025, so we replace
it with **Lambda Function URLs**, which ride on Lambda's permanent free tier at no
extra charge.

---

## Services Used

| Service | Role | Why it's free |
|---|---|---|
| **Android (Kotlin / Jetpack Compose)** | The app UI + local logic | Sideload via USB; no Play Store / developer account needed |
| **Lambda Function URLs** | HTTPS endpoints for the API (no API Gateway) | No per-request fee; only Lambda's always-free tier (1M req + 400k GB-s/mo) applies |
| **AWS Lambda** | `inventoryApi` (GET/POST) + `fridgeWatchdog` compute | Always free: 1M requests + 400,000 GB-seconds per month, forever |
| **Amazon DynamoDB** | Stores inventory items + device tokens | Always free: 25 GB storage + 25 provisioned RCU/WCU (use *provisioned*, not on-demand) |
| **EventBridge Scheduler** | Fires the watchdog on a schedule | Always free: 14M invocations/mo (you use ~30). Supports time zones (JST works directly) |
| **SSM Parameter Store (SecureString)** | Holds FCM service-account key + Gemini API key | Standard tier + default AWS-managed key = free (Secrets Manager is NOT free) |
| **Firebase Cloud Messaging (FCM)** | Delivers push notifications to the phone | Free, unlimited messages |
| **Gemini API (Flash / Flash-Lite)** | Generates the recipe from expiring items | Free tier ~1,500 requests/day, no card required (you use 1/day) |
| **CloudWatch Logs** | Lambda logs | Free at this volume; set 7-day retention so storage can't accumulate |

---

## Architecture

```
                          SYNCHRONOUS (data management)
  +---------------------+        HTTPS + shared-secret header        +----------------------+
  |  Android App (UI)   |  <--------------------------------------> |  Lambda Function URL  |
  |  Jetpack Compose    |   GET /inventory  POST /inventory          |   inventoryApi()      |
  |  Ktor / Retrofit    |   POST /token (FCM registration token)     +----------+-----------+
  +----------+----------+                                                        |
             ^                                                                   v
             | (system notification)                                  +----------------------+
             |                                                         |  DynamoDB            |
  +----------+----------+                                              |  FridgeInventory     |
  |  FirebaseMessaging  |                                              |  (items + tokens)    |
  |  Service (onMessage)|                                              +----------+-----------+
  +----------+----------+                                                         ^
             ^                                                                    |
             | HTTPS POST (FCM v1, OAuth2 token)                                  | Query
             |                                                                    |
  +----------+----------+   reads secrets   +---------------------+   reads/scans +-------------+
  |  Firebase Cloud     | <---------------- |  Lambda             | <-------------+
  |  Messaging endpoint |                   |  fridgeWatchdog()   |
  +---------------------+                   |                     |----> Gemini API (recipe)
                                            +----------+----------+
                                                       ^        \
                          ASYNCHRONOUS (automation)    |         \--- reads keys from
                                            +----------+----------+     SSM Parameter Store
                                            | EventBridge         |     (SecureString)
                                            | Scheduler (cron,JST)|
                                            +---------------------+
```

**Two flows:**
- **Synchronous** — the app calls Lambda Function URLs to read and write inventory.
- **Asynchronous** — EventBridge Scheduler wakes the watchdog on a schedule; it finds
  expiring items, asks Gemini for a recipe, and pushes it to the phone via FCM.

---

## Implementation Steps

### Phase 1 — Database & core compute

1. **Create the DynamoDB table** `FridgeInventory` (provisioned capacity, 25 RCU / 25 WCU
   to stay in the always-free tier).
   - Partition key: `userId` (String)
   - Sort key: `itemId` (String)
   - Store the device token as a reserved item, e.g. `itemId = "DEVICE_TOKEN"`, so you
     don't need a second table.
2. **Store an absolute expiry date, not a day count.** Save `expiryDate` as an ISO-8601
   string (`2026-07-10`) computed from purchase date + shelf life on the client. This lets
   the watchdog compare dates directly instead of doing math on every row.
3. **Write the `inventoryApi` Lambda** (Python or Node.js) handling:
   - `GET /inventory` → `Query` all items for a `userId`
   - `POST /inventory` → `PutItem` with `itemName`, `purchaseDate`, `expiryDate`
   - `POST /token` → store the FCM registration token
4. **Attach an IAM execution role** scoped to *only* this table with `dynamodb:Query`,
   `dynamodb:PutItem`, and `dynamodb:DeleteItem`.

### Phase 2 — Expose the API (no API Gateway)

5. In the Lambda console, open **Configuration → Function URL → Create**.
6. Set **Auth type = NONE** (the URL is a random 32-char string) and add a **shared-secret
   check inside the handler**: the app sends a long random token in a header
   (e.g. `x-app-secret`), and the Lambda rejects any request that doesn't match. Keep the
   secret in SSM (Phase 5), not hard-coded.
   - *This is obscurity + a shared secret, not real auth.* The free "production" upgrade is
     Amazon Cognito (always-free up to 50k monthly active users) with a Lambda authorizer.
7. Test with Postman/Bruno by hitting the Function URL directly with the header set.

### Phase 3 — Android frontend

8. New Android Studio project, **Jetpack Compose (Kotlin)**. Add
   `<uses-permission android:name="android.permission.INTERNET"/>` to `AndroidManifest.xml`.
9. Add **Retrofit** (or Ktor) for networking. Put the Function URL and the shared secret in
   `local.properties` / `BuildConfig`, not in source control.
10. Build the UI:
    - A `LazyColumn` list of items with colored expiry tags (Green = fresh, Red = near expiry),
      computed from `expiryDate`.
    - A floating input modal (item name + date picker) whose Save button issues a `POST`
      to the Function URL.

### Phase 4 — Push notifications

11. Create a free **Firebase project**. Download `google-services.json` into the app's
    `/app` folder and add the Google services Gradle plugin.
12. Implement a class extending `FirebaseMessagingService`; override `onMessageReceived`
    to build a system notification from the incoming data payload.
13. On app launch, capture the **FCM registration token** and send it to `POST /token`
    so the backend knows this phone's address.

### Phase 5 — Secrets (free, via SSM Parameter Store)

14. In **Systems Manager → Parameter Store**, create `SecureString` parameters
    (Standard tier, default `aws/ssm` key — this is free):
    - `/fridge/fcm-service-account` → the Firebase **service-account JSON**
    - `/fridge/gemini-api-key` → your Gemini AI Studio key
    - `/fridge/app-shared-secret` → the header secret from Phase 2
15. Grant each Lambda role `ssm:GetParameter` + `kms:Decrypt` on those parameters only.

### Phase 6 — The watchdog (the brains)

16. **Write the `fridgeWatchdog` Lambda.** It:
    - `Query`/scans `FridgeInventory` for items whose `expiryDate` is within the next N days,
    - bundles those item names into a prompt and calls the **Gemini API** with a system
      instruction like: *"You are an expert chef. Suggest one high-protein recipe using only
      these expiring ingredients."* Use a **Flash / Flash-Lite** model and **do not enable
      billing** on that Google Cloud project (enabling billing removes the free tier).
    - reads the FCM service-account JSON from SSM, **mints an OAuth2 access token**
      (via `google-auth`), and sends an HTTPS POST to the **FCM HTTP v1 endpoint** with the
      recipe. *This OAuth step is the trickiest part of the whole build — budget time for it.*
17. **Fix the schedule/window mismatch:** if the watchdog runs weekly, scan for items
    expiring within **7 days** (not 48 hours), or run it **daily**. A 48-hour window on a
    weekly trigger silently misses items.
18. **EventBridge Scheduler → Create schedule.** Use a cron expression *with the timezone
    field set to `Asia/Tokyo`* so it fires at the local hour you want. Target = the
    `fridgeWatchdog` Lambda.

---

## Two Traps That Break "Free"

1. **Pick the Paid Plan at signup, not the Free Plan.** For accounts created after
   15 July 2025, the *Free Plan* auto-closes after 6 months (or when the $100 credit runs
   out) and shuts down your resources — which would kill the app. The *Paid Plan* only
   charges you when you exceed the always-free tiers, so staying inside them keeps you at
   $0 while the account keeps running. The signup credit just sits there as a buffer.

2. **Set a $1 AWS Budget alert on day one.** AWS does not warn you before a charge lands.
   This is your safety net if you ever provision something outside the free tier.

**Also:** set CloudWatch Logs retention to 7 days per log group, and avoid the services that
are *not* free at rest — API Gateway, Secrets Manager, and NAT Gateway. This design uses
none of them.

---

## Free-Tier Cushion vs. Your Actual Usage

| Resource | Free allowance / month | Your usage | Headroom |
|---|---|---|---|
| Lambda requests | 1,000,000 | a few hundred | >99.9% |
| Lambda compute | 400,000 GB-s | negligible | huge |
| DynamoDB storage | 25 GB | a few KB | huge |
| EventBridge Scheduler | 14,000,000 invocations | ~30 | >99.9% |
| Gemini API (Flash) | ~1,500 requests/day | 1/day | huge |
| FCM messages | unlimited | ~30 | n/a |

*Free-tier terms change — verify current limits on the AWS, Firebase, and Google AI Studio
pricing pages before you build.*
