# Deploying mursense-showcase

This repo is a static site with no build step, dual-hosted:

| Host | URL | Role | Updated by |
|---|---|---|---|
| Cloudflare Pages (project `mursense`) | https://mursense.miao-yu.com | Primary | Git integration: push to `main` auto-builds |
| GitHub Pages | https://ymustc.github.io/mursense-showcase/ | Mirror | push to `main`, always automatic |

DNS: `mursense.miao-yu.com` is a CNAME to `mursense.pages.dev` in the
`miao-yu.com` zone (account ymustc@gmail.com, account ID
`b70081040e3caa2b7075744401a67949`). The separate product demo at
https://mursense-demo.miao-yu.com is a Named Tunnel to a local backend
and has nothing to do with this repo — do not touch it from here.

Content flows one way: `murSense/product-page/` (private working repo
`mursense-upgrade`) is the upstream source of `en/zh/fr.html`; this repo
is a manual copy of it. **Any edit made here must also be made
upstream**, or the next copy silently rolls it back. `index.html` has no
upstream — it lives only here.

## Publishing a change

With Git integration connected (framework preset None, build command
empty, output directory `/`, production branch `main`), one push updates
both hosts:

```bash
git push origin main
```

GitHub Pages is live within a minute or two. Cloudflare clones the repo
and "builds" (a no-op copy) in about the same time. On the Cloudflare
side the trigger is **not** the native push webhook — a project created
via API gets builds but no webhook wiring — it is the GitHub Action in
`.github/workflows/cf-pages-trigger.yml`, which fires the project's
deploy hook (`github-push-main`; its URL lives in the repo secret
`CF_PAGES_DEPLOY_HOOK`, never in the tree) on every push to `main`. If
Cloudflare stops updating on push, check that Action's run log first,
then the hook's existence on the project. Verify against the
servers, not the dashboards — see [Acceptance checks](#acceptance-checks).

### Manual fallback (wrangler direct upload)

If Git integration is ever broken or disconnected, deploy by hand — but
**never from the repo root**. Direct upload takes every file it sees,
including untracked ones: the repo root contains `.claude/worktrees/`
with historical >25MiB videos, and one of those aborts the whole upload
(Cloudflare's per-file limit is 25MiB — the reason the shipped videos
were re-encoded in the first place). Stage exactly what `main` holds:

```bash
STAGE=$(mktemp -d) && git archive main | tar -x -C "$STAGE" && \
/Users/miaoyu/Documents/claudeProjects/miao-yu-lab/node_modules/.bin/wrangler \
  pages deploy "$STAGE" --project-name=mursense --branch=main
```

`--branch=main` is required: without it the upload lands as a preview,
not production, and the custom domain keeps serving the old content.
The pinned wrangler binary above is the one authenticated via OAuth on
this machine.

## Pitfalls already paid for

- **Direct-upload projects cannot be converted to Git integration.**
  Neither the dashboard nor the API allows it — `PATCH` on the project's
  `source` answers error 8000069. What *does* work: `POST` a **new**
  project with the `source` object included (the Cloudflare GitHub App
  was already installed on this account), then move the custom domain
  over. That is how this project got its Git integration.
- **Attaching a custom domain via API does not create the DNS record.**
  The dashboard flow auto-creates the CNAME; the API call registers the
  domain on the project and stops there. Create/verify the CNAME in the
  zone yourself.
- **The wrangler OAuth token has no DNS scope.** It can manage Pages
  projects and deployments but cannot write DNS records; DNS edits go
  through the dashboard or a scoped API token.
- **`cloudflared service install` on macOS installs a LaunchDaemon that
  runs nothing** — twin pitfalls recorded as P0049 in
  `murSense/code/doc/pitfalls/INDEX.md` (demo-tunnel territory, but the
  writeup lives there).
- **25MiB per-file limit** on Cloudflare Pages. `final_en.mp4` and
  `final_fr.mp4` were re-encoded under it (commit `e88db63`); any new
  video must stay under it too.
- **Cloudflare Pages serves "pretty URLs":** `/en.html` answers a 308
  redirect to `/en` with an empty body. A `curl` without `-L` therefore
  sees nothing and a naive grep "proves" a deploy failed when it
  didn't. GitHub Pages serves `.html` paths directly. Always `curl -sL`
  when checking this site.
- **The zone rewrites HTML on the custom domain.** Cloudflare's Email
  Address Obfuscation turns every `mailto:` into a `data-cfemail`
  stub (browsers still see a working link), so the custom domain's
  bytes never match `mursense.pages.dev` or GitHub Pages, and grepping
  the custom domain for `mailto:` "proves" content is stale when it
  isn't — that false alarm cost half an hour on 2026-08-29. Grep for
  surrounding text instead, and run byte-for-byte comparisons against
  `mursense.pages.dev`, which bypasses zone features.

## Acceptance checks

Check the servers, not the consoles. All of these must pass after a
deploy:

```bash
# Every path on the primary domain answers 200 (-L: see the 308 pitfall above)
for p in "" en.html zh.html fr.html final_en.mp4 final_zh.mp4 final_fr.mp4 \
         final_en.srt final_zh.srt final_fr.srt; do
  echo -n "$p: " && curl -sL -o /dev/null -w '%{http_code}\n' "https://mursense.miao-yu.com/$p"
done

# Certificate is valid for the subdomain
openssl s_client -connect mursense.miao-yu.com:443 -servername mursense.miao-yu.com \
  </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate

# Both hosts serve identical content (spot-check the English page).
# pages.dev, not the custom domain: zone-level email obfuscation rewrites
# the custom domain's HTML, so its bytes never match (see pitfalls).
diff <(curl -sL https://mursense.pages.dev/en.html) \
     <(curl -sL https://ymustc.github.io/mursense-showcase/en.html) && echo "hosts in sync"

# The hero CTA actually links to the live demo, and the demo answers
curl -sL https://mursense.miao-yu.com/en.html | grep -c 'mursense-demo.miao-yu.com'
curl -s -o /dev/null -w '%{http_code}\n' https://mursense-demo.miao-yu.com/
```

GitHub Pages can lag a push by a minute or two; a transient diff between
the hosts right after a push is deployment latency, not drift. A diff
that persists means one side missed a deploy — almost always Cloudflare
back when deploys were manual.
