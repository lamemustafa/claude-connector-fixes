# Claude + Salesforce hosted MCP: fix "Authorization failed" and other connector errors (tested Oct 2026)

This guide is for admins who added Salesforce's hosted MCP server to Claude as a custom connector and got an error.

**What we tested.** One free Salesforce Developer Edition org, claude.ai in a browser, a Claude Max account, on 12 October 2026. We did not test sandboxes, production orgs, Claude Team or Enterprise owner flows, or Claude Desktop.

**Labels used on every claim.**

- **CONFIRMED**: we saw it in our own live test on 12 Oct 2026.
- **REPORTED**: someone reported this fix in a public GitHub issue on `anthropics/claude-ai-mcp`. The issue number is given. We did not test it.
- **UNTESTED**: plausible, but nobody has shown it to us.

Claude's error messages are generic. The same message can have more than one cause. Work through the checklist in order.

## 60-second checklist

1. **Check the saved URL.** Open the connector in Claude. Read the saved URL one character at a time. A wrong path gives the same generic errors as a real server problem. (CONFIRMED)
2. **Use your own OAuth client.** The connector must use "Use your own OAuth client" with your External Client App's Consumer Key. Claude pre-selects "Register automatically (DCR)", and Salesforce rejects it. (CONFIRMED)
3. **Turn on JWT-based access tokens.** In the External Client App, tick "Issue JSON Web Token (JWT)-based access tokens for named users". A new app has it off. (CONFIRMED)
4. **Check callback and scopes.** Callback `https://claude.ai/api/mcp/auth_callback`. Scopes "Access Salesforce hosted MCP servers (mcp_api)" and "Perform requests at any time (refresh_token, offline_access)". (CONFIRMED as part of our working setup)
5. **Leave "Require secret" off** for the Web Server Flow and the Refresh Token Flow, unless you also paste the secret in Claude. Our working setup had both off. (CONFIRMED that off works. That "on" breaks it is REPORTED in anthropics/claude-ai-mcp#1070.)
6. **Activate the server** named in your URL: Setup > Integrations > API Catalog > MCP Servers. All servers are inactive in a new org. (CONFIRMED that they start inactive and that our working setup had them active. The error you get when a server is inactive is UNTESTED by us.)
7. **Reconnect after every change.** In Claude, click Disconnect, then Connect. (CONFIRMED)
8. **Read Login History.** Setup > Login History, filtered to your app, around the time you tried. "Failed: ..." means the token exchange failed. "Success" means Salesforce issued a token and the failure came later. (REPORTED in anthropics/claude-ai-mcp#1070 and in a comment on anthropics/claude-ai-mcp#1102. We CONFIRMED that "Success" rows appear while Claude still shows an error.)

## Working baseline (CONFIRMED, 12 Oct 2026)

This exact setup connected and listed tools.

### External Client App

| Setting | Value |
|---|---|
| Callback URL | `https://claude.ai/api/mcp/auth_callback` |
| OAuth scopes | `mcp_api` and `refresh_token` only |
| Require PKCE | On (locked on a new app) |
| Require secret for Web Server Flow | Off (the default) |
| Require secret for Refresh Token Flow | Off (the default) |
| Issue JWT-based access tokens for named users | **On** (default is off) |
| App Authorization / Permitted users | All users can self-authorize |

Where the JWT setting lives: External Client App > Settings > Edit > OAuth Settings > Security.

To copy the Consumer Key, Salesforce asks you to verify your identity with an email code. (CONFIRMED)

### MCP servers

- Setup > Integrations > API Catalog > MCP Servers listed 16 servers in our org. All were Inactive. (CONFIRMED)
- We activated **SObject Reads** and **Headless360 (H360)**. (CONFIRMED)
- H360 is marked Beta. Activating it accepts Salesforce's Beta Services Terms. Read them before you click. (CONFIRMED that the page says so)
- We do not yet know whether a connector to SObject Reads alone works without H360 active. (UNTESTED)

### Claude connector

| Field | Value |
|---|---|
| URL | `https://api.salesforce.com/platform/mcp/v1/platform/headless-360` |
| OAuth client | "Use your own OAuth client" |
| Client ID | The External Client App's Consumer Key |
| Client secret | Empty |

Result: "Connected to Salesforce H360 (test)." Claude listed 5 tools: one interactive, three read-only, one write/delete. All defaulted to "Needs approval". (CONFIRMED)

Hosted MCP works on a free Developer Edition org. (CONFIRMED)

What this baseline does not prove: we saw the tool list, but we did not run a tool call in a chat. (UNTESTED) We did not test the `sobject-reads` or `sobject-all` URLs with the exact path. (UNTESTED)

## Errors, causes and fixes

In each message below, `<name>` is your connector's name. Claude ends most messages with "If this persists, share this reference with support: ofid_...". Keep that reference and the time of the attempt.

### "Couldn't register with `<name>`'s sign-in service"

Full text: "Couldn't register with `<name>`'s sign-in service. You can try again, or add an OAuth Client ID in the connector settings."

- **Cause:** the connector uses Claude's default OAuth client choice, "Register automatically (DCR)". Claude pre-selects it and marks it "Detected" for the Salesforce URL. Salesforce does not register the client. (CONFIRMED)
- **Fix:** remove the connector. Add it again with "Use your own OAuth client" and paste the Consumer Key. (CONFIRMED: with this choice the Salesforce sign-in went through, and our working setup uses it.)
- On Claude Team and Enterprise, only an Owner can add the client ID. (REPORTED in anthropics/claude-ai-mcp#1114)

### "Authorization with `<name>` failed"

Full text: "Authorization with `<name>` failed. You can check your credentials and permissions."

The connector page may also show a **Disconnect** button, so it looks connected, plus "This connector has no tools available."

- **Cause 1: the app issues opaque tokens.** "Issue JSON Web Token (JWT)-based access tokens for named users" is off. (CONFIRMED. We turned it off and got this error. We turned it on and it connected. We did both directions.)
  - **Fix:** External Client App > Settings > Edit > OAuth Settings > Security > tick the JWT setting > Save. In Claude, Disconnect, then Connect. (CONFIRMED)
- **Cause 2: wrong URL path.** We saw this same message once with a mistyped path. JWT was also off in that run, so we cannot say which fault caused it. (Seen, cause UNTESTED) Check the URL anyway.
- **Cause 3: "Require secret" is on.** Login History shows "Failed: Missing Consumer Key Parameter". Fix: untick "Require secret for Web Server Flow" and "Require secret for Refresh Token Flow", keep PKCE. (REPORTED in anthropics/claude-ai-mcp#1070)
- **Cause 4: the Platform Integration User lost `C2CMcpServicePermSet`.** Fix: restore that permission set. (REPORTED in anthropics/claude-ai-mcp#495)
- **Cause 5: the user is not allowed into the app.** The policy is "Admin approved users are pre-authorized" and the user is not assigned. Fix: assign the user through a profile or permission set, or use self-authorize. (REPORTED in forcedotcom/mcp-hosted#12. Anthropic suggested per-user access for new users in anthropics/claude-ai-mcp#839.)
- **Cause 6: wrong scope.** The app has `api` or "Full" instead of `mcp_api`. (REPORTED in anthropics/claude-ai-mcp#184 and anthropics/claude-ai-mcp#1070)

Older wording of the same failure: "Your account was authorized, but the integration rejected the credentials it just issued." Anthropic traced that message to opaque tokens in anthropics/claude-ai-mcp#1093. (REPORTED) We saw the newer wording in October.

### "Your account was authorized, but `<name>` returned an error when connecting"

Full text: "Your account was authorized, but `<name>` returned an error when connecting. You can try again, or check that the server is working."

The connector page then shows "Connection issue. Couldn't connect to the server. Check that the URL points to a valid MCP server."

- **Cause 1: wrong URL path.** We got this exact pair twice, once for `sobject-eads` (missing "r") and once for `headless360` (missing "-"). JWT was on both times. (CONFIRMED)
  - **Fix:** remove the connector and add it again with the exact URL. Then open the connector page and re-read the saved URL. (CONFIRMED that the connector page shows the saved URL.)
- **Cause 2: you used the "Salesforce - Beta" connector and H360 is inactive.** Fix: activate Headless360 (H360). (REPORTED in anthropics/claude-ai-mcp#933 and anthropics/claude-ai-mcp#495)
- **Cause 3: the server is not active for this org, or the user lacks MCP access.** Anthropic traced older reports to Salesforce answering 404 or 405 on the first MCP call. Those reports showed older error wording. (REPORTED in anthropics/claude-ai-mcp#251 and anthropics/claude-ai-mcp#328)

### "Your connection to `<name>` stopped working. Reconnect to continue."

We saw this once right after a successful sign-in. In that run the URL had a typo, the server was inactive and JWT was off. We cannot tell which fault caused it. (Seen, cause UNTESTED) Run the whole checklist.

### OAUTH_APPROVAL_ERROR_GENERIC on the Salesforce approval screen

- **Cause:** the app's scopes are wrong, for example only "Full". (REPORTED in a comment on anthropics/claude-ai-mcp#1070, and in anthropics/claude-ai-mcp#527)
- **Fix:** set the scopes to `mcp_api` and `refresh_token`. (Our working scopes are CONFIRMED. That this clears the approval error is REPORTED.)

### "Couldn't reach the MCP server"

- **Cause:** the saved URL had a stray character at the end, a trailing backtick in one case. (REPORTED in anthropics/claude-ai-mcp#246)
- **Fix:** remove and re-add the connector with a clean URL.

### `invalid_client_id` at test.salesforce.com (sandbox)

- A sandbox with Enhanced Domains is sent to the legacy test.salesforce.com login and fails there. (REPORTED in anthropics/claude-ai-mcp#986, still open, no fix in the thread)
- Sandbox URLs use a `/sandbox/` path segment, and each URL path needs its own Consumer Key. (REPORTED in anthropics/claude-ai-mcp#246 and anthropics/claude-ai-mcp#1114)
- We could not test sandboxes. (UNTESTED)

### Connected, but tool calls fail

- "UNKNOWN_EXCEPTION" on queries: High Assurance session security on the user's profile. Fix: remove it from the profile and grant it through a permission set. (REPORTED in anthropics/claude-ai-mcp#329)
- "Unexpected error while invoking tool" across many users at once: one case was a Salesforce incident. (REPORTED in a comment on anthropics/claude-ai-mcp#782)

### "Authentication required to use this tool", or you sign in again every week

- **Fix:** set the app's Refresh Token Policy to "valid until revoked". (REPORTED by Anthropic in anthropics/claude-ai-mcp#198)

### "A server with this URL already exists"

- Claude allows one custom connector per URL. You cannot add two orgs with the same URL. (REPORTED in anthropics/claude-ai-mcp#614 and anthropics/claude-ai-mcp#178, both open)

## Check the URL first

A wrong path does not give a "not found" message. In our test it gave the same generic errors as a server problem. Salesforce still showed "Success" sign-ins in Login History. (CONFIRMED)

- Production-style paths look like `https://api.salesforce.com/platform/mcp/v1/platform/<server>`. Copy the server URL from the server's detail page in Setup. (CONFIRMED for H360)
- Paths are case-sensitive. (REPORTED in a comment on anthropics/claude-ai-mcp#556)
- Watch for a trailing space or backtick. (REPORTED in anthropics/claude-ai-mcp#184 and anthropics/claude-ai-mcp#246)
- If you remove a connector and add it again, re-read the saved URL. One report says Claude brought back the old record with the old URL. (REPORTED in anthropics/claude-ai-mcp#556)

## You cannot edit the OAuth client after adding

The connector's ⋮ menu offers only "Remove". You cannot switch the OAuth client choice or the client ID later. (CONFIRMED) To change it, remove the connector and add it again. Anthropic's maintainers gave the same advice in anthropics/claude-ai-mcp#1114. (REPORTED)

## "Enable MCP Service" in Anthropic's doc

Step 1 of Anthropic's Salesforce connector doc says to turn on "Enable MCP Service". We could not find a setting by that name in our org in October 2026. Setup search and the User Interface page showed nothing for "MCP Service". Activation happened per server, under API Catalog > MCP Servers. (CONFIRMED for our Developer Edition org. Other editions: UNTESTED.)

The same doc uses the callback `http://127.0.0.1:53280/callback`. That one is for Claude Desktop managed configuration. For a connector added on claude.ai, our working callback was `https://claude.ai/api/mcp/auth_callback`. (CONFIRMED)

## The Flex Credits banner

After you activate a server, the MCP Servers page shows: "You might be billed for Salesforce MCP server usage. Learn about Flex Credits." (CONFIRMED) We do not know what a Developer Edition or production org is charged. (UNTESTED) Check your Salesforce contract before a wide rollout.

## Still stuck?

ComplyEaze fixes these for a fixed fee, no fix no fee: Tapish Khandelwal, tapish@complyeaze.com
