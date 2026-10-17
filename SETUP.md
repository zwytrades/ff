# groupsimple — Discord-connected website + bot

This package contains the updated static website and a Cloudflare Worker that securely connects it to Discord. The site can remain on GitHub Pages; the Worker handles Discord OAuth, listing submissions, moderation buttons, and the bot's slash commands. The design makes the server directory the main focus and shows full descriptions on large cards.

## What it does

- A top-of-page **Join Discord** button. Add your real invite link in `config.js`.
- Discord OAuth sign-in. Visitors can browse without logging in; signing in is required to submit a server.
- No Telegram/X contact-handle field. Moderators can see the submitter's Discord identity in the private request channel. Follow-up instructions tell users to join the groupsimple Discord and message the team.
- Larger server cards that show the server description, network, type, platform, and join link.
- Submissions go to a private Discord moderation channel with **Approve** and **Reject** buttons.
- Approved listings and images are saved into `groups.json` and `pfp/` in your website GitHub repo.
- Bot commands: `/groups`, `/add`, `/remove`, `/boost`, `/unboost`, and `/post`.
- The bot has no always-running process. Discord sends interactions to the Cloudflare Worker, so you do not need a VPS.

## 1. Publish the website

1. Create or use a GitHub repository with GitHub Pages enabled.
2. Upload the **contents** of this package to the repository root: `index.html`, `config.js`, `groups.json`, `banner.png`, `logo.png`, `wrangler.toml`, and the `worker/` folder. `pfp/` will be created when the first image is approved.
3. In the repository, open **Settings → Pages** and publish from branch `main`, folder `/(root)`.
4. Write down the full site URL, including the repository path, for example `https://YOURNAME.github.io/REPO/`. That is your `SITE_URL`.
5. Write down the origin only, without the repository path, for example `https://YOURNAME.github.io`. That is your `ALLOWED_ORIGIN`.

If you use a custom domain, use that domain for both values (the full page URL for `SITE_URL`, and just its origin for `ALLOWED_ORIGIN`). If your existing `groups.json` refers to pictures that are not in the repository, the cards will fall back to server initials until those images are restored; newly approved uploads are stored automatically.

## 2. Create the Discord application and bot

1. Open [Discord Developer Portal](https://discord.com/developers/applications) and create an application named `groupsimple`.
2. On **General Information**, copy the **Application ID** and **Public Key**.
3. On **OAuth2**, copy or create the **Client Secret**. This is a secret and must never go in `config.js` or the GitHub repo.
4. On the **Bot** page, create/reset and copy the bot token. Keep it private.
5. Under **OAuth2 → Redirects**, add the exact callback URL after the Worker has been deployed, in this format: `https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/auth/callback`.
6. In **OAuth2 → URL Generator**, select `bot` and `applications.commands`. Give the bot these permissions in the install flow: **View Channels**, **Send Messages**, **Embed Links**, **Attach Files**, and **Read Message History**. Install it into the Discord server you use for groupsimple.
7. Create a private moderator channel such as `#listing-requests`. Allow the bot to view and send messages there.
8. Enable Developer Mode in Discord (**User Settings → Advanced**). Right-click the private moderator channel and copy its channel ID. Right-click your Discord server and copy the server ID. If you want `/post` to use a default public channel, also copy that channel ID.

## 3. Create a GitHub token

1. In GitHub, open **Settings → Developer settings → Personal access tokens → Fine-grained tokens**.
2. Create a token with access to **only your website repository**.
3. Grant **Contents: Read and write** permission and generate the token. Treat it like a password.

The Worker needs this to update `groups.json` and upload/delete server pictures after moderator actions.

## 4. Deploy the Cloudflare Worker

1. Open [Cloudflare Dashboard](https://dash.cloudflare.com) → **Workers & Pages** → create a Worker.
2. You can use the dashboard editor and paste all of `worker/worker.js`, or deploy from this package with Wrangler (`wrangler.toml` is included). For CLI deploy, run `npx wrangler login` and then `npx wrangler deploy` from this directory.
3. In the Worker, open **Settings → Variables and Secrets**. Add the following values. Mark credentials/tokens as **Secret**.

| Variable | Value | Secret? |
|---|---|---|
| `DISCORD_PUBLIC_KEY` | Application Public Key | Yes |
| `DISCORD_APP_ID` | Application ID | No |
| `DISCORD_CLIENT_SECRET` | OAuth2 Client Secret | Yes |
| `DISCORD_BOT_TOKEN` | Bot token | Yes |
| `DISCORD_CHANNEL_ID` | Private moderator channel ID | No |
| `DISCORD_GUILD_ID` | Your Discord server ID | No |
| `DISCORD_LISTINGS_CHANNEL_ID` | Optional default channel ID for `/post` | No |
| `GITHUB_TOKEN` | Fine-grained GitHub token | Yes |
| `GITHUB_REPO` | `YOURNAME/REPO` | No |
| `GITHUB_BRANCH` | `main` | No |
| `SITE_URL` | Full GitHub Pages URL, e.g. `https://YOURNAME.github.io/REPO/` | No |
| `ALLOWED_ORIGIN` | Site origin only, e.g. `https://YOURNAME.github.io` | No |
| `SESSION_SECRET` | Long random secret, at least 32 characters | Yes |
| `SETUP_KEY` | Random alphanumeric setup key used once to register commands | Yes |
| `ADMIN_USER_IDS` | Optional comma-separated Discord user IDs of additional moderators | No |

`DISCORD_PUBLIC_KEY`, `DISCORD_APP_ID`, and `DISCORD_CLIENT_SECRET` are different values. Generate `SESSION_SECRET` and `SETUP_KEY` yourself; do not reuse your Discord or GitHub tokens. Keep all secrets in the Cloudflare dashboard, never in `config.js`, `index.html`, or GitHub.

## 5. Connect Discord interactions and register commands

1. Copy your Worker address, for example `https://groupsimple.YOUR-SUBDOMAIN.workers.dev`.
2. Return to Developer Portal → **General Information** and set **Interactions Endpoint URL** to `https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/interactions`. Discord should verify it. The Worker verifies Discord's request signature before processing interactions.
3. Open this URL once in your browser, replacing the placeholders with your actual Worker URL and `SETUP_KEY`:

   `https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/register-commands?key=YOUR_SETUP_KEY`

   A successful result lists the registered commands. Do not share the setup URL; after registering commands, you can remove `SETUP_KEY` from Worker variables if you will not need to re-register them. To register them again later, set a new key and update the URL accordingly.

4. In the Discord Developer Portal → **OAuth2 → Redirects**, ensure this exact URI is present: `https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/auth/callback`. The redirect URL must match exactly.

## 6. Connect the website to your Worker and Discord invite

Edit `config.js` in your website repo:

```js
window.GS_CONFIG = {
  workerUrl: "https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev",
  discordInvite: "https://discord.gg/YOUR_REAL_INVITE_CODE"
};
```

- `workerUrl` must be your deployed Worker URL, without a trailing slash.
- `discordInvite` must be your real Groupsimple Discord invite. The package intentionally leaves this blank because no official invite link was included with the source files.

Commit your changes and wait for GitHub Pages to publish, then reload the page.

## 7. Test the whole flow

1. Open the site and press **Login with Discord**. Approve the `identify` permission.
2. Submit a test server with a valid invite, description, and optional icon.
3. Confirm the request appears in `#listing-requests` with Approve and Reject buttons.
4. Click **Approve** as a moderator. It should add the listing to `groups.json`, copy the icon into `pfp/`, and update the moderation message.
5. Wait for GitHub Pages to publish the commit, then refresh the directory.
6. Try the slash commands below in your Discord server.

## Bot commands

- `/groups` — list servers and their IDs.
- `/add` — add a server directly from Discord, including its description, invite, network, and type.
- `/remove group_id` — remove a listing and try to clean up its picture.
- `/boost group_id days` — boost a listing for 1–90 days. Use only after payment is confirmed. Adding days extends an active boost.
- `/unboost group_id` — end a boost early.
- `/post group_id channel` — publish a server as an embed with a **Join server** link in the selected channel. If you configure `DISCORD_LISTINGS_CHANNEL_ID`, the channel option is optional.

Commands and Approve/Reject actions require the Discord **Manage Server** permission or a user ID included in `ADMIN_USER_IDS`. To add multiple IDs, separate them with commas.

## Notes and troubleshooting

- Discord login sessions expire after 12 hours. Only the short-lived, signed groupsimple session token is held in that browser tab; the Discord OAuth access token is not stored in the website.
- A server is not published until a moderator approves it. GitHub Pages may take a minute or two to show a newly committed listing or image.
- If the login returns to the site but does not sign in, check `DISCORD_CLIENT_SECRET`, `SESSION_SECRET`, `SITE_URL`, and the exact Discord OAuth redirect URI.
- If Discord does not validate `/interactions`, check `DISCORD_PUBLIC_KEY` and Worker logs.
- If submission fails, check the bot's access to `DISCORD_CHANNEL_ID`, `GITHUB_TOKEN`, `GITHUB_REPO`, and the site's exact `ALLOWED_ORIGIN`.
- If moderation or commands fail to update the directory, check the Worker logs and confirm the GitHub token has repository Contents read/write permission.
- The public site can be viewed without a login; Discord login is required to submit a listing.
- For stronger spam protection, add a Cloudflare rate-limiting rule for `/submit`. The Worker performs input validation and Discord OAuth authentication but does not promise one request per person.
- Cloudflare Workers Free is sufficient for a small site/bot workload within the current plan limits; verify current usage in your Cloudflare dashboard.
