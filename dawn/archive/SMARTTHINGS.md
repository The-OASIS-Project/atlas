# SmartThings Integration

## Current Status

**OAuth flow is returning 403 Forbidden** from SmartThings authorize endpoint.

## What We've Tried

### 1. App Types Created

We created multiple SmartThings apps via CLI:

| App Type | App ID | Result |
|----------|--------|--------|
| API_ONLY | `0de0135b-1062-4eea-a92e-741fd6de70c0` | 403 Forbidden |
| API_ONLY (new) | `a207ad75-515f-40fb-9a95-c07acd39537f` | 403 Forbidden |
| WEBHOOK_SMART_APP | `2f8f7137-aaf7-4f70-a5e7-559c48f880bd` | 403 Forbidden |

### 2. OAuth Configuration

Current working app (WEBHOOK_SMART_APP):
```json
{
  "appName": "dawn-voice-assistant",
  "appId": "2f8f7137-aaf7-4f70-a5e7-559c48f880bd",
  "appType": "WEBHOOK_SMART_APP",
  "displayName": "DAWN Voice Assistant",
  "webhookSmartApp": {
    "targetUrl": "https://localhost:3000/smartthings/webhook",
    "targetStatus": "PENDING"
  }
}
```

OAuth settings:
```json
{
  "clientName": "DAWN",
  "scope": ["r:devices:*", "w:devices:*", "x:devices:*"],
  "redirectUris": [
    "https://192.168.1.159:3000/smartthings/callback",
    "https://localhost:3000/smartthings/callback"
  ]
}
```

Current credentials in `secrets.toml`:
```toml
[secrets.smartthings]
client_id = "00ef64bc-4b16-4ddf-ac80-ade55cc3d39e"
client_secret = "6c1b2a16-36f0-4565-a492-c0d12dd432b7"
```

### 3. Issues Identified

1. **Redirect URI mismatch**: Server was hardcoding `localhost` instead of using client's `window.location.origin`. Fixed in `webui_server.c`.

2. **App type confusion**: SmartThings community suggests using "OAuth-In App" type via interactive CLI, but we've tried both API_ONLY and WEBHOOK_SMART_APP.

3. **Webhook verification pending**: The WEBHOOK_SMART_APP shows `targetStatus: "PENDING"` - SmartThings may require webhook verification before OAuth works.

4. **403 at AWS load balancer level**: The 403 comes from `awselb/2.0` (AWS Elastic Load Balancer), suggesting the block happens before reaching SmartThings application logic.

## SmartThings OAuth Flow (Per Documentation)

From [SmartThings Community](https://community.smartthings.com/t/unable-to-register-api-only-app-via-cli-403-forbidden-error/302534):

1. Create app with `smartthings apps:create` - select "OAuth-In App" type
2. Configure:
   - Target URL: Where subscription events are received
   - Scopes: Permissions whitelisted for the app
   - Redirect URI: Where authorization code is sent after user approval

3. OAuth flow:
   ```
   GET https://api.smartthings.com/oauth/authorize?
     client_id=${clientId}&
     response_type=code&
     redirect_uri=${redirect_uri}&
     scope=${scopes}
   ```

4. Exchange code for token:
   ```bash
   curl -X POST "https://api.smartthings.com/oauth/token" \
     -u "${clientId}:${clientSecret}" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=authorization_code&client_id=${clientId}&code=${code}&redirect_uri=${redirect_uri}"
   ```

5. Token refresh (access token expires in 24 hours, refresh token in 29 days):
   ```bash
   curl -X POST "https://api.smartthings.com/oauth/token" \
     -u "${clientId}:${clientSecret}" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=refresh_token&client_id=${clientId}&refresh_token=${refresh_token}"
   ```

## SmartThings CLI Commands Used

```bash
# List apps
smartthings apps --json

# Get OAuth settings
smartthings apps:oauth <app-id> --json

# Update OAuth settings
smartthings apps:oauth:update <app-id> -i oauth-settings.json --json

# Regenerate OAuth credentials
smartthings apps:oauth:generate <app-id> -i oauth-gen.json --json

# Create new app
smartthings apps:create -i app.json --json
```

## Personal Access Token (PAT) Alternative

As of December 30, 2024, SmartThings PATs expire after 24 hours (previously up to 50 years).

Generate at: https://account.smartthings.com/tokens

Can be used in `secrets.toml`:
```toml
[secrets.smartthings]
access_token = "your-pat-here"
```

## Findings from Home Assistant Integration

Home Assistant has a working SmartThings integration. Key differences from our approach:

### 1. Centrally-Managed OAuth Credentials

Home Assistant uses **pre-registered OAuth credentials** through their cloud account linking service. Users don't create their own SmartThings apps - HA has credentials already approved by SmartThings.

This explains why their OAuth works but ours returns 403 - their app is "approved" at SmartThings level.

### 2. Different Token Endpoint

HA uses a different token endpoint:
- **Authorization**: `https://api.smartthings.com/oauth/authorize` (same as us)
- **Token exchange**: `https://auth-global.api.smartthings.com/oauth/token` (we use `api.smartthings.com`)

### 3. Comprehensive Scopes

Home Assistant requests more scopes than we do:
```
r:devices:*, w:devices:*, x:devices:*    # Device operations (we have these)
r:hubs:*                                   # Hub access
r:locations:*, w:locations:*, x:locations:* # Location management
r:scenes:*, x:scenes:*                     # Scene control
r:rules:*, w:rules:*                       # Rule management
sse                                        # Server-sent events
r:installedapps, w:installedapps           # Installed app management
```

## Webhook Verification Requirements

For WEBHOOK_SMART_APP type apps, the webhook **must be verified** before the app is fully functional.

### CONFIRMATION Lifecycle

When you create a webhook SmartApp, SmartThings sends a CONFIRMATION event to your webhook:

```json
{
  "lifecycle": "CONFIRMATION",
  "confirmationData": {
    "appId": "{YOUR_APP_ID}",
    "confirmationUrl": "{CONFIRMATION_URL}"
  }
}
```

**Required action**: Make an HTTP GET request to the provided `confirmationUrl`. The URL is only valid for a short time.

### Triggering Re-verification

If you missed the initial verification or it expired:

```bash
curl --request PUT \
  --url https://api.smartthings.com/apps/${APP_ID}/register \
  --header "Authorization: Bearer ${PAT_TOKEN}" \
  --data '{}'
```

### Why This Matters

Our app shows `targetStatus: "PENDING"` which indicates the webhook hasn't been verified. **OAuth may not work until the webhook is verified.**

## Root Cause Analysis

The 403 Forbidden error likely occurs because:

1. **Self-registered OAuth apps may require approval** - Unlike PAT-based access, OAuth apps intended for end-users may need SmartThings approval before they can use the authorize endpoint.

2. **Webhook verification is incomplete** - For WEBHOOK_SMART_APP type, the `targetStatus: "PENDING"` indicates SmartThings hasn't verified our webhook endpoint.

3. **AWS WAF blocking** - The 403 comes from `awselb/2.0` (AWS Elastic Load Balancer), suggesting SmartThings' infrastructure blocks unauthorized OAuth attempts at the WAF level before they reach application logic.

## Recommended Next Steps

1. **Use PAT-based approach** (simpler but less elegant):
   - Generate a PAT at https://account.smartthings.com/tokens
   - Store in `secrets.toml` as `personal_access_token`
   - Tokens now expire after 24 hours (changed December 2024)
   - Would need token refresh mechanism or user re-authentication

2. **Implement webhook verification** (if continuing with OAuth):
   - Add `/smartthings/webhook` endpoint handler
   - Handle CONFIRMATION lifecycle events
   - Make GET request to `confirmationUrl` to verify
   - Check if `targetStatus` changes from "PENDING"
   - Re-test OAuth flow

3. **Contact SmartThings developer support**:
   - Ask about OAuth app approval process for 3rd party apps
   - Confirm whether webhook verification is required for OAuth

4. **Consider SmartThings Edge drivers**:
   - Alternative integration method
   - Runs locally on SmartThings Hub
   - No OAuth needed

## Related Files

- `src/tools/smartthings_service.c` - SmartThings API client
- `src/webui/webui_server.c` - WebSocket handlers for SmartThings messages
- `www/js/admin/smartthings.js` - WebUI SmartThings module
- `secrets.toml` - OAuth credentials storage

## References

- [SmartThings Developer Documentation](https://developer.smartthings.com/docs/getting-started/authorization-and-permissions)
- [SmartThings CLI GitHub](https://github.com/SmartThingsCommunity/smartthings-cli)
- [API App Subscription Example](https://github.com/SmartThingsCommunity/api-app-subscription-example-js)
- [Community: 403 Forbidden Issues](https://community.smartthings.com/t/unable-to-register-api-only-app-via-cli-403-forbidden-error/302534)
