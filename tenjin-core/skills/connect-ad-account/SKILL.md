---
name: connect-ad-account
description: Use when the user wants to connect, add, create, or set up an ad-network account, channel, channel integration, or ad network in Tenjin — e.g. "create a channel", "create an ad network", "create a channel integration", "add an ad account", "connect Tapjoy", "set up Facebook ads". Covers resolving the right ad network and handing the user the dashboard link to finish the connection.
---

# Connecting an Ad Account in Tenjin

Use this skill when the user wants to connect an ad network — also called a
"channel" or "ad account" — to their Tenjin account.

You cannot create the ad account yourself, and you should not try. Connecting
an ad network means storing its credentials (an API key, a certificate, or an
OAuth grant), and Tenjin only accepts those through its own session-authenticated
web dashboard — there's no API for it, by design. Your job is to find the right
ad network and hand the user a link; the dashboard handles the rest.

## Steps

1. **Identify the ad network.** Take the name from the user's request (Tapjoy,
   Facebook, AppLovin, etc.). If they didn't name one, ask which network they
   want to connect.

2. **Resolve its id.** Call `list_channels` with `query` set to the network
   name.
   - **No match:** call `list_channels` again with no query to list the
     available networks, and ask the user to pick one.
   - **Multiple matches:** list the candidates (name and id) and ask which one
     they mean — don't guess.

3. **Hand off to the dashboard.** Tell the user to log in to the Tenjin
   dashboard and open this URL, with the resolved id filled in:

   `https://dashboard.tenjin.com/dashboard/integrations/new?ad_network_id={id}`

   For example, Tapjoy (id 1):
   `https://dashboard.tenjin.com/dashboard/integrations/new?ad_network_id=1`

   Let them know the dashboard is where they'll enter the network's
   credentials to finish the connection.

## Checking what's already connected
Connecting happens in the dashboard, but you can read the result: `list_ad_accounts`
(filter by `query` or `channel_id`) shows the ad accounts already connected. Use it to
check whether an account exists before sending the user off, or to confirm a connection
finished. Each ad account's `id` is the `ad_account_id` used when creating a campaign
against that specific account.

## Credentials are off-limits here

Never collect, request, echo, or store an ad-network credential (API key,
secret, certificate, private key, password, OAuth token), and never start an
OAuth flow yourself. This isn't a style preference: credential storage is
architecturally confined to Tenjin's dashboard, so there's nothing safe for
the agent or MCP server to do with one anyway.

If the user pastes a credential into the chat, don't repeat it back — just
point them to the dashboard link from step 3 and continue from there.

## Output style

Lead with the dashboard link for the network you resolved. Keep the reply
short: which network you matched, the link, and a one-line note that they'll
finish the connection there.
