---
name: framer-deploy
description: This skill should be used when deploying a Framer site to a custom domain through Vercel, especially with a Cloudflare-managed domain and Framer's reverse-proxy/self-hosting pattern.
---

# Framer Deploy

## Overview

Deploy a Framer-hosted site behind a Vercel project so a custom domain can serve the Framer output while keeping Vercel as the public host/CDN. Use this for requests like "deploy my Framer site to example.com on Vercel", "self-host Framer with a reverse proxy", "put this Framer project behind Cloudflare + Vercel", or "deploy a Framer site without paying Framer for custom-domain/branding removal".

Core principle: publish Framer first, proxy Vercel to the stable Framer origin, verify the real public domain, hide Framer branding only when explicitly requested, then record the exact deployment details.

## When to Use

- Deploying a Framer project to a custom apex or `www` domain through Vercel.
- Moving a Framer site from `*.framer.website` to a public domain managed in Cloudflare.
- Fixing a Vercel deployment that serves a local shell instead of the Framer site.
- Repeating the Vercel proxy deployment pattern for another Framer site.
- Using the "free proxy pattern" where Vercel serves a `*.framer.website` origin on a custom domain and hides the visible Framer badge via HTML CSS injection.

Skip this skill for ordinary Framer design edits, non-Framer Vercel apps, or DNS-only changes with no reverse proxy.

## Required Inputs

Identify or ask for:

- Framer project ID or existing Framer project/session.
- Framer production origin, ideally `https://<site>.framer.website`.
- Public domain, including apex and `www` expectations.
- Vercel project name and team scope.
- DNS provider access only if DNS is missing or incorrect.

Do not commit or push unless the user explicitly grants permission in the current conversation.

## Step 1: Check Current Context

Run a quick local inspection before changing anything:

```bash
pwd
git status --short
find . -maxdepth 3 -name vercel.json -o -path '*/.vercel/project.json'
```

If working inside a larger project workspace, follow that workspace's logging or handover rules before and after deployment.

## Step 2: Verify Current Docs

Use current official docs when available because hosting rules can change:

- Framer reverse-proxy/self-hosting guide.
- Vercel domains guide.
- Vercel routing or project-configuration docs.

Preserve these Framer constraints unless official docs have changed:

- Use a custom domain or `*.framer.website` as the Framer origin.
- Do not proxy to `*.framer.app`.
- Avoid trailing slashes in proxied public URLs.
- Do not mutate Framer's `<head>` or main HTML payload.
- Set the Framer canonical/public URL to the final public domain when the plan exposes that setting.

## Step 3: Publish Framer

Install/setup the Framer agent if needed:

```bash
npx @framer/agent@latest setup
npx @framer/agent@latest session new "<framer_project_id>"
```

Preview before publishing:

```js
framer.agent.publish({ action: "preview" })
```

If the preview returns warnings or errors, resolve them before publishing unless the user explicitly accepts the risk.

Confirm publish using the returned hash:

```js
framer.agent.publish({
  action: "confirm_publish",
  confirmationHash: "<confirmation_hash>"
})
```

Record:

- Framer project ID.
- Framer origin URL.
- Published version ID.
- Any warnings, errors, or manual follow-ups.

## Step 4: Check Vercel Project and Domains

Inspect the linked Vercel project:

```bash
cat .vercel/project.json 2>/dev/null || true
vercel project inspect "<project_name>" --scope "<team_scope>"
vercel domains inspect "<domain>" --scope "<team_scope>"
vercel domains inspect "www.<domain>" --scope "<team_scope>"
```

Check DNS:

```bash
dig +short "<domain>" A
dig +short "www.<domain>" A
dig +short "www.<domain>" CNAME
```

Expected Vercel DNS defaults:

- Apex: `A 76.76.21.21`
- Subdomain: `CNAME cname.vercel-dns.com`

Existing working records can be left alone if Vercel verifies and serves the domain correctly. Change Cloudflare only when Vercel reports invalid configuration or the DNS clearly points elsewhere.

## Step 5: Configure the Reverse Proxy

Prefer a front-door route when the project also contains local files or build output. Plain `rewrites` can lose to filesystem routing and accidentally serve an old local app shell.

Use this `vercel.json` shape:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "trailingSlash": false,
  "routes": [
    {
      "src": "/(.*)",
      "dest": "https://<site>.framer.website/$1"
    }
  ]
}
```

Use `https://<site>.framer.website` with no trailing slash in `dest` before `/$1`.

If the project has no local output and official Vercel docs confirm external rewrites are enough, `rewrites` can be acceptable. Verify live HTML before calling it done.

## Step 6: Deploy to Vercel

Run local validation/build checks if the project has them:

```bash
npm run build
```

Deploy production:

```bash
vercel deploy --prod --yes --scope "<team_scope>"
```

Inspect the final deployment and aliases:

```bash
vercel inspect "https://<domain>" --scope "<team_scope>"
```

Record:

- Deployment ID.
- Deployment URL.
- Live aliases.
- Whether any earlier deploy served stale/local output and how it was corrected.

## Step 7: Verify Public Behaviour

Use HTTP checks against the real public domain:

```bash
curl -I -L "https://<domain>"
curl -L "https://<domain>" | head -n 20
curl -I -L "https://www.<domain>"
curl -I "https://<domain>/about/"
curl -I -L "https://<domain>/contact"
```

Confirm:

- Status is `200` for apex and `www`.
- HTML contains Framer output, often including `Made in Framer`.
- Page title and key H1 match the expected site.
- Trailing slash redirects behave as expected, usually `308` to the non-trailing path.
- Important routes return Framer HTML, not a stale local app shell.

Run browser QA with desktop and mobile viewports when the site is visual or user-facing. Capture screenshots and check console errors. Record screenshot paths in the venture log.

## Step 8: SEO and Framer Follow-Up

Check canonical/social metadata:

```bash
curl -L "https://<domain>" | rg -i 'canonical|og:url|twitter:url|framer.website'
```

If metadata still points at `*.framer.website`, report it as a Framer-side setting. Set the canonical/public URL in Framer to `https://<domain>` if the plan exposes the setting.

Also report visible Framer badges if they remain; remove those through Framer plan/settings, not by mutating proxied HTML.

### Free Proxy Pattern: Hide the Framer Badge

Use this when the user explicitly wants the practical free route after being told the clean Framer plan/settings route. In this pattern, Vercel serves the custom domain, Framer remains the content origin, and a Vercel Edge Function injects CSS to hide the visible badge.

Treat this as a workaround, not a Framer-supported billing/settings feature:

- Expect Framer selectors or badge behaviour to change in the future.
- Expect this to depend on Vercel Function availability and usage limits.
- Keep the implementation obvious and easy to remove if the site later moves to a paid Framer plan.
- Do not present it as legally equivalent to an official Framer plan feature.

Prefer adding CSS through the Vercel proxy instead of deleting Framer markup:

- Keep Framer's `#main` payload untouched.
- Do not strip scripts, tags, or badge HTML from the response.
- Inject a single `<style>` tag into the HTML response:

```css
#__framer-badge-container,
.__framer-badge {
  display: none !important;
  visibility: hidden !important;
  pointer-events: none !important;
}
```

If the current Vercel config is a pure external route, replace it with a small Edge Function proxy that fetches the Framer origin, passes non-HTML assets through unchanged, injects the CSS only into `text/html`, and sets a diagnostic header such as `x-framer-proxy: badge-hidden`.

Use this file shape for a TypeScript Vercel project:

```text
api/framer.ts
api/framer/[...path].ts
src/server/framerProxy.ts
vercel.json
```

In `vercel.json`, route the homepage and all paths to the local proxy functions:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "trailingSlash": false,
  "routes": [
    {
      "src": "/",
      "dest": "/api/framer"
    },
    {
      "src": "/(.+)",
      "dest": "/api/framer/$1"
    }
  ]
}
```

In the proxy implementation:

- Build the upstream URL from the incoming request path and query.
- Fetch from `https://<site>.framer.website`.
- Remove request headers that can break proxying, especially `host` and `accept-encoding`.
- Remove hop-by-hop response headers, especially `content-length` and `content-encoding`, because HTML length changes after injection.
- Return non-HTML responses unchanged.
- For `text/html`, inject the badge-hiding style before `</head>`.
- Set a diagnostic response header such as `x-<project>-framer-proxy: badge-hidden`.

Verify with Playwright that `#__framer-badge-container` computes to `display: none`, has zero dimensions, and does not create console errors on desktop or mobile.

## Step 9: Handover

Finish with:

- Live domain and `www` status.
- Framer version and origin.
- Vercel deployment ID and URL.
- Config file path changed.
- Verification performed.
- DNS changes made or explicitly not needed.
- Remaining manual Framer settings.
- Confirmation that no git commit/push was made unless explicitly authorised.
