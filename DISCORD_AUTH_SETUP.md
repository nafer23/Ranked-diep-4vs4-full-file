# Discord OAuth2 Login Setup

This server supports "Login with Discord". Users click a button, get redirected
through Discord's OAuth2 flow, and come back with a signed ticket that acts as
their password for the WebSocket Init packet.

## 1. Create a Discord Application

1. Go to <https://discord.com/developers/applications> and click **New Application**.
2. Under **OAuth2 → General** note down:
   - **Client ID**
   - **Client Secret** (click *Reset Secret* if needed)
3. Under **OAuth2 → Redirects** add your callback URL:
   - Local dev: `http://localhost:8080/auth/discord/callback`
   - Production: `https://your-domain/auth/discord/callback`

The callback URL the server uses must match *exactly* what Discord has on file.

## 2. Provide credentials to the server

You can provide credentials either via environment variables or by creating an
`auth.config.json` file in the project root. The config file takes priority.

### Option A: `auth.config.json` (Recommended)

Create `auth.config.json` and fill in your details:

```json
{
  "clientId": "123456789012345678",
  "clientSecret": "your-secret-here",
  "authSecret": "64-hex-characters-for-signing",
  "publicUrl": "http://localhost:8080", // Set this to your domain. The redirectUri will be derived from it.
  "guildId": "optional-guild-id",
  "requiredRoleIds": ["optional-role-id"],
  "requireAuth": true
}
```

### Option B: Environment Variables

Set these variables before starting the server:

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `DISCORD_CLIENT_ID` | yes | `123456789012345678` | From the Developer Portal |
| `DISCORD_CLIENT_SECRET` | yes | `abc...` | From the Developer Portal |
| `AUTH_SECRET` | recommended | `openssl rand -hex 32` | HMAC key used to sign tickets. If unset, a random value is picked per boot (all tickets are invalidated on restart). |
| `PUBLIC_URL` | recommended in prod | `https://diep.example.com` | Used to build the redirect URI. If unset, it is derived from the incoming `Host` header (fine for `localhost`). |
| `DISCORD_REDIRECT_URI` | optional | `https://diep.example.com/auth/discord/callback` | Only set this if you need a non-standard callback path, or if `PUBLIC_URL` is not sufficient. |
| `DISCORD_GUILD_ID` | optional | `123456789012345678` | If set, server requires the user be a member of this guild and reads their roles. See §3b. |
| `DISCORD_REQUIRED_ROLE_IDS` | optional | `111..,222..` | Comma-separated. If set, user must have at least one of these roles. |

### PowerShell (local dev)

```powershell
$env:DISCORD_CLIENT_ID = "123456789012345678"
$env:DISCORD_CLIENT_SECRET = "your-secret-here"
$env:AUTH_SECRET = "replace-with-64-hex-chars"
npm run dev
```

### Docker

```
docker run ... \
  --env DISCORD_CLIENT_ID=... \
  --env DISCORD_CLIENT_SECRET=... \
  --env AUTH_SECRET=... \
  --env PUBLIC_URL=https://diep.example.com \
  diepcustom
```

## 3. Grant elevated access to specific Discord users (optional)

By default, logged-in users get `AccessLevel.PublicAccess`. To grant yourself
`BetaAccess` (god mode / level up / switch tank) or `FullAccess`, edit
`src/config.ts`:

```ts
export const discordAccessLevels: Record<string, AccessLevel> = {
    "YOUR_DISCORD_USER_ID": AccessLevel.FullAccess,
    "FRIEND_DISCORD_USER_ID": AccessLevel.BetaAccess,
};
```

To find your Discord user ID: in Discord, *User Settings → Advanced → Developer
Mode*, then right-click your own name → *Copy User ID*.

Rebuild (`npm run build`) after editing.

## 3b. Role-gated login from a specific Discord server

You can require users to be in a particular server and optionally have a
particular role (e.g. "Verified") before they're allowed to play, and map
Discord roles to in-game access levels.

### Step 1 — Get the guild (server) ID and role IDs

1. Enable Developer Mode in Discord: *User Settings → Advanced → Developer Mode*.
2. Right-click the server icon → **Copy Server ID** → this is your guild id.
3. Server Settings → Roles → right-click each role → **Copy Role ID**.

### Step 2 — Configure

Set env var:

```
DISCORD_GUILD_ID=123456789012345678
```

**Optional — required role (gate):** user is rejected if they don't have at
least one of these roles. Use this for a "Verified", "Member", etc. gate.
Comma-separated:

```
DISCORD_REQUIRED_ROLE_IDS=987654321098765432,234567890123456789
```

**Optional — role to access-level mapping:** edit `src/config.ts`:

```ts
export const discordRoleAccessLevels: Record<string, AccessLevel> = {
    "ROLE_ID_VERIFIED":    AccessLevel.PublicAccess, // normal players
    "ROLE_ID_BETA_TESTER": AccessLevel.BetaAccess,   // cheats allowed
    "ROLE_ID_STAFF":       AccessLevel.FullAccess,   // dev-level
};
```

The user's effective level is the **maximum** among the roles they possess. A
per-user override in `discordAccessLevels` still wins over this map.

### Step 3 — Let users know what'll happen

When `DISCORD_GUILD_ID` is set, the OAuth2 consent screen will now ask the user
to approve both:

- *Access your username and avatar* (`identify`)
- *See your roles in* **Your Server** (`guilds.members.read`)

If they uncheck the second one, the login is denied.

### What users get rejected for

| Case | HTTP status | Shown message |
|---|---|---|
| Not a member of `DISCORD_GUILD_ID` | 403 | "You are not a member of the required Discord server." |
| In the guild but missing every `DISCORD_REQUIRED_ROLE_IDS` role | 403 | "Your Discord account doesn't have the required role in the server." |
| Unchecked `guilds.members.read` on the consent screen | 403 | "You must grant the 'See your server roles' permission..." |

Rebuild (`npm run build`) after editing `config.ts`.

## 4. Flow summary

1. User opens `/` and clicks **Login with Discord** (top-right).
2. Browser is redirected to `/auth/discord/login` → Discord's authorize page.
3. Discord redirects back to `/auth/discord/callback?code=...&state=...`.
4. Server verifies the `state`, exchanges `code` for an access token, fetches
   the user's identity (`identify` scope only), signs an `AuthTicket`, and
   redirects to `/?token=<ticket>`.
5. The snippet in `client/index.html` stores the ticket in
   `localStorage.password` and strips it from the URL.
6. The WASM client uses that value as the `pw` field on the Init packet.
7. `Client.ts` verifies the HMAC signature and TTL, then assigns the access
   level declared inside the ticket.

Tickets are valid for 7 days by default (`authTicketTtlMs` in `config.ts`).

## 5. Troubleshooting

- **503 "Discord login is not configured"** — `DISCORD_CLIENT_ID` or
  `DISCORD_CLIENT_SECRET` is empty.
- **"Invalid OAuth2 state"** — the `state` cookie expired (>10 min) or the user
  opened `/auth/discord/callback` directly. Start again from
  `/auth/discord/login`.
- **Discord returns `redirect_uri_mismatch`** — the URL this server computes
  doesn't match what you registered in the Developer Portal. Either register
  both `http://localhost:8080/auth/discord/callback` and your prod URL, or set
  `PUBLIC_URL` / `DISCORD_REDIRECT_URI` explicitly.
- **Restarting the server logs everyone out** — set a persistent `AUTH_SECRET`.
- **User was granted the wrong access level** — edit `discordAccessLevels` in
  `src/config.ts` and rebuild. The user will need to log out and log back in to
  receive a new ticket.
