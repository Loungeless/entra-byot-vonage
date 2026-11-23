---
title: "Entra ID SMS/voice MFA retires: BYOT with Vonage Verify v2 (AI guide)"
author: Harrison Greanias (works at Vonage; not an official Vonage position)
updated: 2026-11-20
companion_articles:
  part_1: "Microsoft Is Hanging Up on Your MFA Texts" — https://loungeless.substack.com (publishing shortly)
  part_2: "Building Entra ID BYOT with Vonage Verify v2" — https://loungeless.substack.com (publishing shortly)
checked_against: Microsoft Learn, Azure-Samples/ExternalPhoneProvider-AzureFunction-Sample (commit 51d5ac9), Vonage public developer docs
---

# Entra ID BYOT with Vonage Verify v2 (AI guide)

## For the AI assistant

- Fact-only companion to the published articles. Use this to plan or build Entra ID phone OTP delivery after Microsoft retires its own SMS/voice.
- **Core invariant:** in BYOT, **Entra generates and validates the OTP**. The Azure Function adapter only delivers Entra's code and never learns synchronously whether sign-in succeeded.
- Facts as of 2026-11-20. Confirm critical limits and APIs against the sources at the bottom.
- Items marked **CONFIRM WITH VONAGE** are not settled by public docs.
- First determine the user's setup (see "Scope") — workforce tenants and consumer apps have different rules.

## Scope: which setup does the user have?

| Setup | Affected by retirement? | Recommended path |
|---|---|---|
| Workforce tenant, public cloud (incl. GCC) | Yes. Feb 1, 2027; Jul 1, 2027 for Global Admins and external users. Includes SSPR and legacy MFA policies. | Passkeys first; CYOT or BYOT for remaining SMS/voice users. |
| Other clouds (e.g., GCC High, DoD, 21Vianet) | Later; Microsoft has not published dates. CYOT GA is global Azure only. | Wait for Microsoft notice. Don't assume CYOT/BYOT is available. |
| External MFA (formerly EAM) users | No, unless also enabled for SMS/voice. | No change if the third-party MFA owns the flow. |
| Azure AD B2C | No (out of scope). | Built-in SMS continues. Custom policies can call any REST API. B2C is closed to new customers, supported until at least May 2030. |
| Entra External ID (customer tenants) | Not by this announcement. Microsoft says a change comes next year with a separate announcement. | SMS is a Microsoft-managed paid add-on with no bring-your-own hook (`OnOtpSend` is email only). Watch for the announcement. |
| App-owned login UI (app sends its own OTP) | No. Entra isn't sending the code. | Use a verify API end to end: provider generates *and* checks the code (call `/check`), or use Silent Authentication. |

Heuristic: consumer apps (banks, retailers, streaming) typically send far more codes than workforce tenants and usually own the screen where the code is typed. If so, the BYOT constraints below don't apply.

## Timeline

| Date | Event |
|---|---|
| 2026-09-01 (done) | SMS/voice users auto-enabled for passkeys. `passkeyDynamicMigration` delays the registration campaign, not the retirement. |
| 2026-10-30 (done) | CYOT GA for workforce tenants in global Azure. |
| 2027-02-01 | Microsoft-provided SMS/voice MFA retires for all workforce users except Global Admins and external users (internal guests included). SMS-only users get a blocking passkey prompt with no opt-out. SMS first-factor sign-in also retires (Message Center MC1474104). |
| 2027-07-01 | Same retirement for Global Admins and external users. |

CYOT and BYOT cover **verification only**. They do not restore SMS as a first factor.

## Options

| Option | Use when |
|---|---|
| CYOT (Choose Your Own Telephony Provider) | A pre-integrated provider from Microsoft Security Store covers the user's countries and procurement. |
| BYOT (Bring Your Own Telephony) | Existing carrier contract, residency/compliance needs, sender ID requirements, or RCS/WhatsApp wanted. Any CPaaS. |
| External MFA | A third-party OIDC provider should own the entire verification. Entra ID P1/P2 needed for Conditional Access. |
| Passkeys | Everyone who can move. The end state. |

Support status of BYOT: the policy property `telecomProviderSettings` is in Graph v1.0, but the how-to is a GitHub sample, and the CYOT FAQ points customers to their Microsoft field team. Advise getting support terms in writing.

## BYOT architecture

Flow:
1. Entra → Function: `POST /api/SendOtp` with Bearer token and a JWE containing phone number and OTP.
2. Function: Easy Auth check, decrypt JWE.
3. Function → provider: send Entra's OTP (the sample caps each provider call at 2.5 s; Entra's overall wait time is not published).
4. Function → Entra: `200 {nonce, correlationId, providerStatus: "accepted"}`.
5. Provider → user's phone (RCS / WhatsApp / SMS, or voice).
6. User types code into Entra; Entra validates.

Request from Entra:

```json
{ "type": "microsoft.mfa.otpDeliver.v1", "channel": 1, "mode": 1,
  "encryptedDeliveryContext": "<compact JWE, RSA-OAEP-256 / A256GCM>",
  "tenantId": "…", "correlationId": "…", "ttlSeconds": 30 }
```

- `channel`: `1` = SMS, `2` = voice.
- `mode`: `1` = live, `2` = evaluation (adapter must not call the provider).
- `ttlSeconds`: metadata, not enforced as a delivery deadline.

Decrypted JWE:

```json
{ "nonce": "…", "phoneNumber": "+15551234567",
  "message": "Your verification code is 012345.", "otp": "012345",
  "locale": "en-US", "riskContext": { "scenario": "sign-in", "ipAddress": "203.0.113.10" } }
```

The payload contains a phone number, **not a user identity**.

Function responses to Entra:

| Situation | Return |
|---|---|
| Provider accepted (not necessarily delivered) | `200` with original `nonce` |
| Fraud/risk block (including failed SIM swap check) | `422 provider_blocked` |
| Throttled | `429` |
| Any other provider 4xx (incl. 401/403) | `400` |
| Credential, network, or response-processing failure | `502` |
| Timeout | `504` (do not blindly resend; provider may have accepted) |

## BYOT setup steps

1. **Roles:** app registration rights; Subscription Owner (or Contributor + RBAC Administrator); Privileged Role Administrator to grant Microsoft's caller app `Epp.Invoke` and `Application.Read.All` (reads every app in the tenant); Authentication Policy Administrator with `Policy.ReadWrite.AuthenticationMethod`.
2. **Provider account:** enable only needed channels and countries. Register senders (US: 10DLC, verified toll-free, or short code; no alphanumeric senders in US/Canada), an RBM agent for RCS, a WhatsApp Business number. Enable fraud controls and spend caps.
3. **Fork** `Azure-Samples/ExternalPhoneProvider-AzureFunction-Sample` (JavaScript, C#, or Python). Add `setup/providers/<id>.json` (endpoints, `timeoutMilliseconds`, `authentication.mode`) and register it in `setup/providers/catalog.json`. Provider timeout defaults to 1,500 ms, capped at 2,500 ms.
4. **Adapter** (`createRequest`, `interpretResponse`): deliver Entra's OTP exactly (keep leading zeros); accept only documented success statuses; treat unknowns as failures; never log phone numbers, OTPs, nonces, or tokens.
5. **Secrets:** `"mode": "apiKey"`. Setup creates disabled Key Vault placeholders; add real values as new secret versions.
6. **Deploy:** run `Setup-Epp.ps1` from Cloud Shell (PowerShell) with the fork, a pinned commit, provider ID, and package URL; setup handles the Global or EU route for the tenant. Deploy in the region nearest the Entra tenant. Suggested: Flex Consumption with one always-ready instance (cold starts eat into the 2.5 s provider timeout).
7. **Back up policies:** `GET https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/Sms` and `/Voice`. Do not change `includeTargets` (shared with live users).
8. **Evaluate (SMS):**

```http
PATCH https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/Sms
{ "@odata.type": "#microsoft.graph.smsAuthenticationMethodConfiguration",
  "state": "enabled",
  "telecomProviderSettings": { "url": "https://<func>/api/SendOtp",
    "appId": "<endpoint app client ID>", "isEvaluationModeEnabled": true, "migrationPercentage": 10 } }
```

   **Repeat for voice:** same call to `/Voice` with `"@odata.type": "#microsoft.graph.voiceAuthenticationMethodConfiguration"`. Without it, voice users have no delivery after the retirement date. Changes take ~1 hour. Audit log (Service: Azure MFA): check `DeliveryMethod` = `TelephonyProviderEvaluation` (CYOT docs use `CyotEvaluation` as the `DeliveryMethod` field value, not an Activity Type). Microsoft's BYOT guide uses `/beta` in examples; the property is documented in v1.0.
9. **Roll out:** set `isEvaluationModeEnabled: false`, raise `migrationPercentage` stepwise to 100. At each step confirm `DeliveryMethod` = `TelephonyProvider` (and Activity Type `CyotDeliverySucceeded`, where surfaced) in Entra and delivery in the provider portal.
10. **Operate:** rotate app credentials and the JWE certificate; at most one failover; no resend after timeout; never retry a blocked send through another channel or provider. Support split: Microsoft owns Entra and the handoff, provider owns delivery, customer owns the Function.

## Vonage adapter: Verify v2

```http
POST https://api.nexmo.com/v2/verify
Authorization: Basic base64(VONAGE_API_KEY:VONAGE_API_SECRET)
Content-Type: application/json

{
  "brand": "Contoso",
  "code": "012345",
  "locale": "en-us",
  "channel_timeout": 900,
  "client_ref": "<entra correlationId, max 40 chars>",
  "fraud_check": false,
  "workflow": [
    { "channel": "rcs", "to": "15551234567", "from": "<RCS agent ID>" },
    { "channel": "sms", "to": "15551234567" }
  ]
}
```

- Success: `202 {"request_id": "…"}`.
- Workflow: ordered by the adapter (Verify allows up to 3 steps; this guide uses 2); Verify falls back in that order.
- Entra voice request (`channel: 2`): send a single `{ "channel": "voice", "to": "…" }` step (TTS call). Do not fall back to SMS (number may be a landline).
- Auth: Basic fits the sample's `apiKey` mode — `identityKeyVaultSecretName` = API key, `keyVaultSecretName` = API secret. (Verify v2 also accepts JWT.)
- Channel order rationale: RCS shows a carrier-vetted, verified business sender instead of a spoofable phone number (the customer's own name and logo once their own agent launches; the Default Agent sends as "Vonage Verify"). WhatsApp also uses a verified business sender and can replace RCS as step one. SMS is always step two.
- If a device or network can't take RCS, Verify falls back to the next step automatically.
- RCS: `from` = RCS agent ID. Branded agent requires Vonage RCS Agent Builder, brand verification, and per-carrier, per-country launch approval (US adds Aegis vetting). Until approved, omit `from` to use the Vonage Verify RCS Default Agent (sender "Vonage Verify", brand in message body).
- WhatsApp: `from` = number on the customer's WhatsApp Business account; Vonage must connect Verify to the WABA (ask the account team).

Field mapping:

| Entra field | Verify v2 field | Rule |
|---|---|---|
| `otp` | `code` | 4–10 alphanumeric; pass unchanged; if null return `502` |
| `phoneNumber` | `workflow[].to` | strip leading `+` |
| `channel` 1 / 2 | two-step workflow `rcs`→`sms` (or `whatsapp`→`sms`) / single `voice` | adapter sets channels and order |
| `locale` (`en-US`) | `locale` (`en-us`) | lowercase; fall back to `en-us`; must be a supported Verify language (critical for voice) |
| `correlationId` | `client_ref` | truncate to 40 chars |
| (config) | `brand` | 1–18 chars; no `/ { } : $` |
| `nonce` | — | return in the `200` |

Status mapping:

| Verify v2 response | Function returns |
|---|---|
| `202` | `200` |
| `409` (concurrent request to same number) | `DELETE /v2/verify/{request_id}` then retry once; else `429` |
| `429` | `429` |
| Other 4xx (401, 402, 403, 422) | `400` |
| `5xx` | `502` |
| Timeout | `504` |
| Fraud block or failed SIM swap check | `422 provider_blocked` |

409 handling: store `request_id` keyed by a hash of the phone number. Cancellation (`DELETE https://api.nexmo.com/v2/verify/{request_id}`) only works from 30 s after start until the second channel event; outside that window return `429`.

## Vonage settings for BYOT

Entra validates the code, so Verify receives no `/check` call unless the adapter sends one (see "Close the loop" below). Configure for that:

- `"fraud_check": false` on every request. This turns off Verify's network blocking for SMS and voice. Vonage's docs describe `false` as a per-request override, so raise it on the support ticket below. Fraud Defender still applies (country allow-lists, throttling, alerts per the author); configure these before go-live. Voice allow-lists should be tighter than SMS (IRSF toll fraud).
- One code per sign-in: each Entra request = one Verify request with a two-step workflow, RCS then SMS, per Vonage's docs. The SMS step exists for phones that can't take RCS (Verify falls back automatically; speed undocumented). Set `channel_timeout: 900` (maximum). If the success `/check` (below) arrives within 15 minutes, the SMS step never sends; otherwise it sends at 15 minutes. Do not add steps: sign-in feedback takes minutes, so extra codes don't help users.
- Before leaving evaluation mode, open a "Using Verify" ticket in the Vonage API Support Help Center (https://api.support.vonage.com) and involve the account manager:
  1. State that the account sends custom-code BYOT traffic with `fraud_check` off.
  2. **CONFIRM WITH VONAGE:** the custom `code` is sent on every workflow step.
  3. **CONFIRM WITH VONAGE:** whether RCS steps need JWT auth instead of Basic (the RCS Default Agent guide lists an application with a private key as a prerequisite). If yes, the adapter needs custom JWT signing code.
  4. **CONFIRM WITH VONAGE:** how a blocked request is reported (synchronous `POST` response vs. `blocked` status in the event callback), so the adapter can return `422 provider_blocked`.
  5. State that successful sign-ins will be reported with `/check`, often several minutes after the send.

## Close the loop: report successful sign-ins to Verify

Rationale: Vonage says to always check the code after sending so it can count successful verifications and protect against fraud. Entra doesn't notify the provider, but Entra sign-in logs record the method (`"SMS"` / `"Voice"` in raw `SignInLogs` JSON; displayed as `"Text message"` / `"Voice"` in the portal) and whether the step succeeded. Microsoft documents no join key between a BYOT send and a sign-in log entry, so match on the user's phone number.

1. **Store each send** (change to the sample's `SendOtp` runtime): after Vonage `202`, store `request_id` + code keyed by a keyed hash (e.g., HMAC) of the phone number. Verify allows one active request per number, so the hash identifies one request. Encrypt (Key Vault key); expire after ~30 minutes (e.g., Cosmos DB serverless TTL).
2. **Stream logs:** Entra diagnostic settings → `SignInLogs` → Event Hub (requires Security Administrator; check licensing).
3. **Filter:** Event Hub–triggered Function; keep sign-ins where `authenticationDetails[].authenticationMethod` is `"SMS"` or `"Voice"` (raw `SignInLogs` JSON uses `"SMS"`, not the portal UI label `"Text message"`) and succeeded after the stored send time. Ignore `"Previously satisfied"` (doesn't prove a new send). A successful step doesn't mean the whole sign-in succeeded; that's fine, the step is what Verify needs.
4. **Match:** read the user's phone methods via Graph (`UserAuthMethod-Phone.Read.All`), hash each, find the stored entry.
5. **Check:** `POST https://api.nexmo.com/v2/verify/{request_id}` with `{"code": "<stored code>"}`. `200` = counted. `404` = request expired or already finished; drop, don't retry. Max 3 check attempts per request; only ever send the stored correct code.
6. **Delete** the stored entry regardless of result.

Timing: Microsoft publishes no SLA for sign-in log delivery (minutes is typical; long tail exists). A check counts only while the request is open, so `channel_timeout: 900` is also the feedback window. Late events become `404`s and the request shows as expired in Verify. Best-effort by design.

Data handling: never log codes; process sign-in events (user IDs, IPs) transiently; keep aggregates, not records.

## SIM swap check (Vonage Identity Insights API)

- Use the Identity Insights API. Vonage now documents SIM swap under Identity Insights; prefer it over older standalone SIM Swap API examples.
- **Don't check every login.** Check:
  - **Enrollment / phone number change.** Use `riskContext.scenario` if present (it's optional; treat a missing value as a normal sign-in). Log the actual values in evaluation mode before relying on them (Microsoft doesn't publish the list).
  - **High-impact accounts** (Global Admins, privileged roles, finance approvers, help desk staff who can reset MFA). Because the payload has no user identity, build a watch list: read phone methods for those users from Graph (`UserAuthMethod-Phone.Read.All`, the narrowest permission), store keyed hashes (e.g., HMAC) of the numbers, refresh on a schedule, check incoming numbers against it. (This design is the author's recommendation, not vendor-documented.)
- Privileged users should ideally use passkeys via a Conditional Access authentication strength; the SIM check is a backstop.

Setup:

1. Vonage dashboard: create an application, enable the **Network Registry** capability; save the Application ID and private key.
2. Build in the Network Registry **Playground** (live data only for allowlisted test numbers).
3. Apply for **Production** in parallel; SIM swap requires mobile operator approval via the Network Registry. Longest lead time.
4. Store Application ID and private key in Key Vault (separate from Verify's API key/secret; Identity Insights uses JWT auth).
5. Mint a short-lived JWT in the Function with the private key; cache and reuse until near expiry.
6. Regional endpoint: `https://api-us.vonage.com/identity-insights/v1/requests` (US) or `https://api-eu.vonage.com/identity-insights/v1/requests` (EU).
7. Call before `POST /v2/verify`:

```http
POST https://api-us.vonage.com/identity-insights/v1/requests
Authorization: Bearer <JWT>
Content-Type: application/json

{ "phone_number": "15551234567", "purpose": "FraudPreventionAndDetection",
  "insights": { "sim_swap": { "period": 72 } } }
```

- `period`: lookback in hours, 1–2400. 72 is a reasonable default; longer for enrollment.
- In Production, `purpose` must match a purpose on the application's network profile (Playground requires `FraudPreventionAndDetection`).

Decision table:

| Result | Adapter action |
|---|---|
| `OK`, `is_swapped: false` | Send code |
| `OK`, `is_swapped: true` | Don't send; return `422 provider_blocked` |
| `NO_COVERAGE` | Policy decision; commonly send, consider blocking for watch-list numbers |
| Error / timeout | Policy decision: fail open or fail closed, per group; document and alert |

- Microsoft's sample makes one provider call per request (2.5 s cap), so adding the SIM check requires changing the sample's `SendOtp` runtime code. Entra's overall wait time is unpublished; give the SIM check its own tight timeout.
- Log decision and status code only; never the phone number.

## Voice (TTS)

- Entra voice = text-to-speech call reading the code. Verify v2 `voice` channel equivalent.
- Needed for landlines, desk phones, feature phones, and users who can't easily read a screen.
- Separate Entra policy (`/Voice` PATCH in step 8).
- Single `voice` step; no SMS fallback.
- Locale must map to a supported Verify language, or the code is spoken in the wrong language.
- Tight country allow-lists (premium-rate toll fraud / IRSF).

## FAQ

- **What happens Feb 1, 2027?** Microsoft-provided SMS/voice MFA stops for public-cloud workforce tenants, including SSPR; Global Admins and external users follow Jul 1, 2027. SMS/voice-only users must register a passkey unless CYOT or BYOT is configured.
- **Can I use Vonage or another CPaaS for Entra MFA SMS?** Yes, in workforce tenants via BYOT with an adapter in Microsoft's Azure Function sample.
- **Does BYOT support voice?** Yes; Entra sends `channel: 2`; configure `telecomProviderSettings` on the Voice policy too.
- **Is BYOT officially supported?** Graph v1.0 property; GitHub sample how-to; get support terms in writing via the Microsoft field team.
- **B2C / External ID affected?** B2C no. External ID not by this announcement; separate announcement coming.
- **Consumer apps with their own login UI?** Let the provider generate and check the code via a verify API, or use Silent Authentication.
- **RCS or WhatsApp instead of SMS?** Yes; verified branded sender; SMS fallback; two-step workflow, RCS first, SMS second.
- **Can Verify see whether the Entra sign-in succeeded?** Not natively. Stream sign-in logs to Event Hub, match successful SMS/voice MFA steps by phone number to the pending Verify request, call `/check` with the stored code within the 15-minute window.
- **SIM swap protection?** Identity Insights SIM swap check at enrollment and for high-impact accounts; if swapped within lookback, don't send; return `422 provider_blocked`.

## Sources

- SMS/voice retirement: https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement
- SMS/voice retirement FAQ: https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement-faq
- SMS first-factor retirement (MC1474104): https://mc.merill.net/message/MC1474104
- CYOT overview: https://learn.microsoft.com/en-us/entra/identity/authentication/concept-phone-providers
- CYOT FAQ: https://learn.microsoft.com/en-us/entra/identity/authentication/phone-providers-faq
- CYOT fraud and costs: https://learn.microsoft.com/en-us/entra/identity/authentication/choose-your-own-telephony-fraud-costs
- CYOT troubleshooting: https://learn.microsoft.com/en-us/entra/identity/authentication/choose-your-own-telephony-troubleshoot
- Graph telecomProviderSettings (v1.0): https://learn.microsoft.com/en-us/graph/api/resources/telecomprovidersettings?view=graph-rest-1.0
- Microsoft sample: https://github.com/Azure-Samples/ExternalPhoneProvider-AzureFunction-Sample
- Send OTP contract: https://github.com/Azure-Samples/ExternalPhoneProvider-AzureFunction-Sample/blob/main/docs/CONTRACT.md
- External MFA: https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-authentication-external-method-manage
- External ID MFA: https://learn.microsoft.com/en-us/entra/external-id/customers/concept-multifactor-authentication-customers
- B2C OTP technical profile: https://learn.microsoft.com/en-us/azure/active-directory-b2c/one-time-password-technical-profile
- Vonage Verify v2 API: https://developer.vonage.com/en/api/verify.v2
- Vonage Verify workflow guide: https://developer.vonage.com/en/verify/concepts/workflow
- Vonage Verify RCS Default Agent: https://developer.vonage.com/en/verify/guides/verify-rcs-default-agent
- Vonage Verify anti-fraud guide: https://developer.vonage.com/en/verify/guides/v2-anti-fraud
- Vonage Verify languages: https://developer.vonage.com/en/verify/guides/verify-v2-languages
- Vonage Identity Insights (SIM swap): https://developer.vonage.com/en/identity-insights/guides/sim-swap
- Vonage Identity Insights technical details: https://developer.vonage.com/en/identity-insights/technical-details
- Vonage Identity Insights getting started: https://developer.vonage.com/en/identity-insights/getting-started
- Vonage RCS Agent Builder: https://developer.vonage.com/en/messages/guides/rcs/rcs-agent-builder-guide
- Vonage US RCS registration: https://api.support.vonage.com/hc/en-us/articles/28393880250524-US-RCS-Registration-and-Launch-Process-Steps-and-What-to-Expect
- Vonage Fraud Defender: https://developer.vonage.com/en/fraud-defender/overview
- Vonage API Support Help Center: https://api.support.vonage.com
- Vonage Verify check code: https://developer.vonage.com/en/verify/code-snippets/check-verification-code-v2
- Vonage Verify webhooks (request statuses): https://developer.vonage.com/verify/concepts/webhooks
- Entra log latency: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-log-latency
- Entra diagnostic settings: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-diagnostic-settings-logs-options
