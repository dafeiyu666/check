# GAME UPDATE CHECK

**语言 / Language:** [中文](README.md) ・ **English**

---

## About

An game version update checker, extracted from an existing production build, running entirely on **Cloudflare Workers** in JavaScript. It supports version checks for:

- Game for Peace (和平精英)
- BETA PUBGM
- PUBGM
- BGMI

## How it works

- **Web read**: reads `.apk` information from a web page to get the latest version
- **Local comparison**: keeps the current version locally and compares it against the detected latest version
  - Detected version **lower than** current → nothing is pushed
  - Detected version **higher than** current → an update is pushed

## Tech Implementation (Cloudflare Worker)

The whole logic runs on a **Cloudflare Worker** — pure JavaScript, no server of your own required:

- **Runtime**: the Worker executes JS, uses `fetch` to grab the `web.gpubgm.com` pages, then extracts `.apk` / `.ipa` download links and version numbers with regular expressions
- **Scheduled trigger**: Cron Triggers run automatic checks for Android + iOS
- **Manual trigger**: visit the Worker's `/android` or `/ios` path to run a check immediately
- **Storage**: Cloudflare KV (binding name `PUBG_KV`) stores the current version and download links, powering the "local comparison" (see KV Configuration below)
- **Push**: when a new version is found, a Telegram Bot `sendMessage` pushes it to your channel / topic
- **Source**: [`beta/beta.js`](beta/beta.js) (BETA PUBGM beta), [`hp/hp.js`](hp/hp.js) (Game for Peace beta) and [`bgmi/bgmi.js`](bgmi/bgmi.js) (BGMI India)

> Before use, replace `TG_BOT_TOKEN`, `TG_CHAT_ID` (channel / group ID) and `TG_THREAD_ID` (topic ID; use `0` when no topic) at the top of `beta/beta.js` with your own values (see the tutorials below).

## Tutorial 1: Create a Bot & Get the Token via @BotFather

1. Search for **@BotFather** in Telegram (official, blue-verified) and open it
2. Send `/newbot`
3. Enter a **display name** for your bot (anything, e.g. `Game Update Check`)
4. Enter a **username** (must be globally unique and end with `bot`, e.g. `game_update_check_bot`)
5. BotFather replies with a **token**, like:

   ```
   123456789:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

   That is your `TG_BOT_TOKEN` — keep it secret. If it ever leaks (e.g. pushed to a public repo), send `/revoke` to BotFather immediately to regenerate it
6. Useful commands: `/mybots` (manage your bots), `/token` (view the token again), `/setprivacy` (group privacy mode)

## Tutorial 2: Get the Group ID and Topic ID (`TG_CHAT_ID` / `TG_THREAD_ID`)

1. Add your bot **to your group** with permission to send messages (making it an admin is recommended)
2. Send a test message in the group — inside the target topic if topics are enabled
3. Open the following URL in a browser (replace `<YOUR_TOKEN>` with the token from Tutorial 1):

   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```

   If the list is empty, send `/start@your_bot_username` in the group and refresh
4. In the JSON response, look for:
   - `"chat": { "id": -100xxxxxxxxxx }` → this is **`TG_CHAT_ID`**
   - `"message_thread_id": 4` → this is **`TG_THREAD_ID`** (topic ID; use `0` if there is no topic or you don't need one)
5. Another way to find the topic ID: copy the link of any message inside the topic, e.g.:

   ```
   https://t.me/c/1234567890/4/123
   ```

   The middle number (`4`) is the topic ID

> Pushing to a **channel** also works: add the bot as a channel admin, post a message, and the `chat.id` in `channel_post` from `getUpdates` is your `TG_CHAT_ID` (channels usually have no topics — set `TG_THREAD_ID` to `0`).

## KV Configuration (Cloudflare KV)

Before deploying, create a **KV namespace** in Cloudflare and bind it to every Worker (**the binding variable name must be `PUBG_KV`** — all three scripts access `env.PUBG_KV`):

1. Cloudflare dashboard → **Workers & Pages → KV** → **Create namespace** (any name, e.g. `pubg-kv`)
2. Open each Worker → **Settings → Variables and Bindings → KV Namespace Binding**: set the variable name to `PUBG_KV` and pick the namespace you just created
3. Save and redeploy

Keys used by each script — **no need to create them manually**; the scripts write them automatically when an update is detected:

### beta/beta.js (BETA PUBGM beta)

| Key | Purpose |
| --- | --- |
| `pubg_exp_current_version` | Android overall version (for comparison) |
| `pubg_exp_32_version` / `pubg_exp_32_url` | Android 32-bit version / download URL |
| `pubg_exp_64_version` / `pubg_exp_64_url` | Android 64-bit version / download URL |
| `pubg_exp_apk_links` | Android 32/64-bit links summary |
| `pubg_ios_current_version` | iOS current version |
| `pubg_ios_ipa_links` | iOS package link |

### hp/hp.js (Game for Peace beta)

| Key | Purpose |
| --- | --- |
| `hpios_current_version` | iOS current version |
| `hpios_ipa_link` | iOS package link |
| `hpandroid_current_version` | Android current version |
| `hpandroid_apk_link` | Android package link |

### bgmi/bgmi.js (BGMI India)

| Key | Purpose |
| --- | --- |
| `bgmi_current_version` | Current version |
| `bgmi_apk_link` | APK download link |

> All three scripts share the binding name `PUBG_KV`: one shared namespace works (the keys don't collide), or you can create a separate namespace per script — both are fine.

## Note

This project uses the "read `.apk` from a web page" approach, not the alternative one — **the alternative we are unable to open-source**.

## Screenshot

![Screenshot](img/jietu.png)
