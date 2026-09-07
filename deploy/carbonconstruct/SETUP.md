# Postiz for CarbonConstruct — setup guide

This folder deploys one self-hosted Postiz instance that publishes to the
CarbonConstruct LinkedIn profile, LinkedIn Page, Facebook Page and Instagram.
CarbonConstruct's admin composer (`/admin/social`) talks to it through the
Postiz Public API; Postiz owns OAuth tokens, scheduling and retries.

Everything below was taken from the provider code in this repository
(`libraries/nestjs-libraries/src/integrations/social/*.provider.ts`) — the
redirect URIs and scopes are what the code actually sends, not a paraphrase of
Postiz' hosted docs.

## 0. What you need before starting

| Item | Why |
|---|---|
| A Linux host with Docker + Docker Compose v2 (2 vCPU / 4 GB RAM is enough) | Runs Postiz, Postgres, Redis, Temporal, Caddy |
| A DNS A record, e.g. `postiz.carbonconstruct.com.au` → host IP | Caddy issues the TLS certificate automatically; every OAuth redirect is HTTPS |
| Ports 80 and 443 open inbound | Let's Encrypt + the app |
| LinkedIn Developer app (section 2) | LinkedIn profile + Page |
| Meta (Facebook) developer app (section 3) | Facebook Page + Instagram |

## 1. Deploy

```bash
git clone https://github.com/stvn101/postiz-app
cd postiz-app/deploy/carbonconstruct
cp .env.example .env
# fill in POSTIZ_DOMAIN, JWT_SECRET (openssl rand -base64 48), POSTIZ_DB_PASSWORD,
# and the provider credentials from sections 2-3
docker compose up -d
docker compose logs -f postiz     # wait for "Nest application successfully started"
```

Open `https://<POSTIZ_DOMAIN>` and register the first account — that account
becomes the organisation owner. Then set `DISABLE_REGISTRATION=true` in `.env`
and run `docker compose up -d` again so nobody else can sign up.

## 2. LinkedIn app

Postiz has two LinkedIn providers. Both use the same app and the same scopes:

```
openid  profile  w_member_social  r_basicprofile
rw_organization_admin  w_organization_social  r_organization_social
```

Because the organisation scopes are always requested, the app must have the
**Community Management API** product approved — LinkedIn rejects the OAuth
request with "unauthorized_scope_error" if any requested scope is not granted
to the app. Plan for that approval up front; it needs the app to be associated
with a verified LinkedIn Company Page (the CarbonConstruct page).

1. Go to <https://www.linkedin.com/developers/apps> → **Create app**. Associate it
   with the CarbonConstruct Company Page and verify the association from the
   Page admin view.
2. **Products** tab → request:
   - *Sign In with LinkedIn using OpenID Connect* (`openid`, `profile`) — instant
   - *Share on LinkedIn* (`w_member_social`) — instant
   - *Community Management API* (`r_basicprofile`, `rw_organization_admin`,
     `w_organization_social`, `r_organization_social`) — application review
3. **Auth** tab → *Authorized redirect URLs for your app*, add both:
   ```
   https://<POSTIZ_DOMAIN>/integrations/social/linkedin
   https://<POSTIZ_DOMAIN>/integrations/social/linkedin-page
   ```
4. Copy **Client ID** and **Primary Client Secret** into `.env` as
   `LINKEDIN_CLIENT_ID` / `LINKEDIN_CLIENT_SECRET`.

Connect in Postiz: **Launches → Add channel → LinkedIn** (profile) and
**LinkedIn Page** (choose the CarbonConstruct page when prompted).

## 3. Meta app (Facebook Page + Instagram)

1. <https://developers.facebook.com/apps> → **Create app** → use case
   *Other* → type **Business**. Link it to the business portfolio that owns the
   CarbonConstruct Facebook Page and Instagram account.
2. Add products **Facebook Login for Business** and **Instagram** (Graph API).
3. **Facebook Login for Business → Settings → Valid OAuth Redirect URIs**:
   ```
   https://<POSTIZ_DOMAIN>/integrations/social/facebook
   https://<POSTIZ_DOMAIN>/integrations/social/instagram
   ```
4. **App settings → Basic**: copy **App ID** / **App secret** into `.env` as
   `FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET`. Set a privacy-policy URL
   (`https://carbonconstruct.com.au/privacy`) — Meta requires one before the app
   can leave Development mode.
5. Permissions the providers request (grant them under *App Review →
   Permissions and features* if Meta asks; while the app is in **Development
   mode** they work for any user who has a role on the app, which covers the
   CarbonConstruct pages):
   - Facebook Page: `pages_show_list business_management pages_manage_posts
     pages_manage_engagement pages_read_engagement read_insights`
   - Instagram (via Facebook Business): `instagram_basic pages_show_list
     pages_read_engagement business_management instagram_content_publish
     instagram_manage_comments instagram_manage_insights`
6. The Instagram account must be a **Business** (or Creator) account **linked to
   the Facebook Page**; Postiz' `instagram` provider walks Page → Instagram
   account.

Connect in Postiz: **Add channel → Facebook Page** (pick the page), then
**Add channel → Instagram (Facebook Business)** (pick the same page).

### Instagram without a Facebook Page (optional)

If the Instagram account is not linked to a Page, use Postiz' *Instagram
(Standalone)* provider instead: add the **Instagram API with Instagram Login**
product to the Meta app, register
`https://<POSTIZ_DOMAIN>/integrations/social/instagram-standalone` as its
redirect URI, and fill `INSTAGRAM_APP_ID` / `INSTAGRAM_APP_SECRET` (the Instagram
app id/secret shown on that product's page, not the Facebook app id).

## 4. Give CarbonConstruct an API key

1. In Postiz, open **Settings → Public API** and copy the API key.
2. In the Supabase dashboard for CarbonConstruct (project `htruyldcvakkzpykfoxq`)
   → **Edge Functions → Secrets**, add:
   ```
   POSTIZ_API_URL = https://<POSTIZ_DOMAIN>/api
   POSTIZ_API_KEY = <the key>
   ```
   The `/api` suffix matters: the public routes live at `<backend>/public/v1/*`
   and Caddy proxies `/api` to the backend.
3. Deploy the CarbonConstruct edge functions `postiz-channels`, `postiz-posts`,
   `postiz-sync` and apply migration `20260906000000_scheduled_social_posts.sql`
   (see `SOCIAL_AUTOMATION.md` in the CarbonConstruct repo).

Verify from any shell:

```bash
curl -s -H "Authorization: <the key>" https://<POSTIZ_DOMAIN>/api/public/v1/integrations | jq
```

You should see one object per connected channel with `identifier` values
`linkedin`, `linkedin-page`, `facebook`, `instagram`. Those are exactly what
`/admin/social` lists.

## 5. Day-to-day

- Compose in CarbonConstruct `/admin/social`: pick channels, write once, post
  now or schedule. Postiz publishes; results flow back within 15 minutes (or
  press *Sync now*).
- Anything Postiz can do that the composer can't (threads/comments, carousels,
  stories, per-channel edits, the calendar view) is available in Postiz itself
  — posts created from CarbonConstruct show up in its calendar too.
- Tokens: LinkedIn tokens last 60 days and Postiz refreshes them; Meta page
  tokens are long-lived. If a channel shows *disabled* in the composer, reconnect
  it in Postiz (**Launches → channel avatar → Reconnect**).

## 6. Upgrades

```bash
cd postiz-app/deploy/carbonconstruct
docker compose pull postiz && docker compose up -d postiz
```

Postiz runs its Prisma migrations on start. Take a `pg_dump` of the
`postiz-postgres` volume first if you want a rollback point.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| LinkedIn: `unauthorized_scope_error` | Community Management API not approved on the app (section 2, step 2). |
| Facebook connect shows no pages | The logged-in Facebook user is not an admin of the Page, or the app has no role for that user while in Development mode. |
| Instagram connect fails "not a business account" | Convert the account to Business/Creator and link it to the Page (section 3, step 6). |
| Composer says "Postiz is not connected" | `POSTIZ_API_URL` / `POSTIZ_API_KEY` secrets missing on the Supabase edge runtime, or the functions were not deployed. |
| `429` from the Public API | Raise `API_LIMIT` in `.env` (requests per hour per key) and `docker compose up -d`. |
| Image upload rejected by Postiz | Postiz downloads the image from a signed Supabase URL; it must be JPG/PNG/WebP under Postiz' size cap and the Postiz host must be able to reach `*.supabase.co`. |
