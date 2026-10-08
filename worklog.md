# Worklog

An honest audit trail. What was built, what was verified, what was deployed, and
what was deliberately not done. Written so a future session can tell the
difference between a thing that is finished and a thing that is merely claimed.

## Credentials stored (all 0600, outside any git repo)

| File | Contents |
|---|---|
| `~/.config/secrets/cloudflare.env` | Cloudflare account id, API token, R2 access key and secret, S3 endpoint |
| `~/.config/secrets/github.env` | Fine-grained PAT (`GITHUB_TOKEN`), classic PAT (`GITHUB_TOKEN_CLASSIC`) |

Both were pasted in plaintext into a chat session, so they are present in
opencode's on-disk transcript at `~/.local/share/opencode/opencode.db`. Rotate
all three credentials (Cloudflare plus both GitHub tokens) if that transcript is
ever shared, synced or backed up.

The classic PAT carries roughly 24 scopes including `admin:org`, `admin:repo_hook`,
`delete_repo` and `workflow`, and has no expiry. It is far broader than this work
needs. It was needed because the fine-grained token returned HTTP 403 on
`POST /gists` and could not push over Git. A narrower replacement with `gist` and
`repo` scopes plus a 30 to 90 day expiry would be sufficient for all work here.

Git is configured to read credentials from `~/.git-credentials-push` (mode 0600)
rather than embedding the token in `.git/config`. Verified that `.git/config`
contains no token and that no tracked file contains any credential value.

## Standing rules added to memory

`~/.config/opencode/AGENTS.md` now carries, permanently across sessions:

- Never use an em dash or an en dash, anywhere, in any file.
- Published promotion content stays positive. Nothing about absent features, nothing
  about release maturity, nothing about audience size.
- Every factual claim must be true; omission is acceptable, falsehood is not.
- `psmux` owns the project repositories and has no access from this environment
  by design; `deepaash` is the planning and promotion account only.
- The DeskVNC project summary, verified metrics and the verified constraints.

## Environment repairs caused by my own earlier change

The "free models only" request broke oh-my-openagent's model routing. All
category and agent targets became unresolvable for two separate reasons:
`enabled_providers` blocked every `openai/*` model, and a `provider.opencode-go`
whitelist cut that provider to two models, deleting `opencode-go/kimi-k3`,
`opencode-go/minimax-m3`, `opencode-go/grok-4.7` and others.

Repaired with 50 substitutions in `~/.omo/omo.jsonc`, mapping every dead target
onto a free model that resolves. Verified all nine referenced models resolve.
Also added the `writing` category, which was never configured and so silently
fell back to the dead `kimi-k3`.

A backup of the original config is at `~/.omo/omo.jsonc.bak.<timestamp>`. A
warning comment now sits in `opencode.jsonc` because the coupling between the
provider whitelist and the plugin's model routing is not obvious and it fails
silently.

## Content built by the swarm

Produced by parallel subagents on free models, each with a disjoint output
directory, all working from `swarm/CONTEXT.md` which holds the verified facts and
the hard rules.

| Area | Files | Words |
|---|---|---|
| Long-form posts | `swarm/posts/` 5 files | 5255, minimum 965 per file |
| Use cases and comparison | `swarm/content/` 6 files | 8113, minimum 1085 per file |
| Boundary support | `swarm/boundary/` 4 files | 2876 |
| Technical deep dives | `swarm/technical/` 6 files | 6514, minimum 921 per file |
| Gist sources | `swarm/gists/` 5 files | 3634 |
| Discovery layer | `swarm/seo/` 6 files | 2230 |
| GitHub Pages hub | `swarm/repo/` 8 files | 2243 |
| RSS plus MCP server | `swarm/dist/` 8 files | 476 |
| Illustrations | `swarm/art/` SVG files | n/a |

Two fabrications were caught and removed rather than shipped. The posts agent had
invented a `STALE_GENERATION` error code and a `LEASE_EXPIRED` error code. Neither
exists; the real codes verified against `skills/deskvnc/SKILL.md` in the project
repository are `LIMB_GONE`, `SCREEN_CHANGED` and `LEASE_REVOKED`, and the real
call for checking takeover is `dvv_control` with `action: "yield_status"`. Both
passages were rewritten. A sweep of the whole tree for the invented codes now
returns zero.

The gists agent also independently removed four unverifiable specifics it had
written: a binary size claim, a release cadence claim, a line count and a named
third-party integration.

The comparison agent verified competitor facts by fetching RustDesk's docs,
Guacamole's manual, and Wikipedia for RealVNC, TigerVNC and mstsc. It correctly
determined RustDesk uses its own protocol rather than speaking VNC despite
carrying a `vnc` GitHub topic.

The MCP server agent read the real `crates/dvv/src/mcp/manifest.rs` rather than
inventing tool signatures. It found that `dvv_clipboard get` and `dvv_term_read`
are documented as not served on the current build, and it did not hide that. It
reframed the terminal guide around `dvv_screen` on a terminal limb, which is the
working path.

## A real bug found in the hub pages

All four hub pages declared a canonical URL of
`https://psmux.github.io/deskvnc/hub/...`. That path returns **HTTP 404**. The
pages actually live on Cloudflare Pages, so the canonical was both broken and
claiming a location on a repository this account does not control. Repointed all
four canonical tags, all four `og:url` tags and all four sitemap locs at
`https://deskvnc-hub.pages.dev/`.

The same fix was applied to the GitHub Pages mirror, which declares the
Cloudflare host as canonical so search authority consolidates instead of
splitting between two identical copies.

## Compliance gate

`scripts/compliance-check.sh` runs seven checks: zero em or en dashes, no negative
framing, no fabricated proof points, no dead anchors or placeholders, no external
resource dependencies, JSON validity, and HTML well-formedness via a tag-stack
parser.

It caught eight violations in my own earlier `site/` output, which was written
before the positivity rule was given. That output framed the product by its
limits, named its maintainer count, and carried a section about where the product
is the wrong choice. All eight were rewritten: the negatives became an affirmative
section about where the product fits, written as five reasons rather than five
caveats.

The checker was itself fixed twice. The forbidden-word pattern matched the
three letter word fragment inside the name of a social network, so the pattern is now word-anchored
with a comment explaining why, because an unanchored future edit would silently
reintroduce the false positive. The link extractor also included trailing sentence
punctuation, producing two false 404s on the issues and releases URLs.

The whole tree passes. Every first-wave directory passes independently.

## Images

Four real project screenshots were taken from the upstream repository README and
recompressed. They are 87 to 95 percent smaller than the originals for
equivalent quality, cutting 3.7 MB of PNG down to 292 KB of JPEG. This is the
same payload problem flagged in the strategy's Phase 1.

They were initially not used at all: the hub pages shipped with zero `<img>` tags.
The images are being integrated now, along with the approved logomark.

## Live surfaces

| Surface | URL |
|---|---|
| Cloudflare Pages (canonical origin) | https://deskvnc-hub.pages.dev/ |
| Pages guides | https://deskvnc-hub.pages.dev/guides |
| Pages use cases | https://deskvnc-hub.pages.dev/usecases |
| Pages benchmarks | https://deskvnc-hub.pages.dev/benchmarks |
| Pages llms.txt | https://deskvnc-hub.pages.dev/llms.txt |
| Pages robots.txt | https://deskvnc-hub.pages.dev/robots.txt |
| Pages feed | https://deskvnc-hub.pages.dev/feed.xml |
| Pages FAQ | https://deskvnc-hub.pages.dev/faq.json |
| Pages structured data | https://deskvnc-hub.pages.dev/structured-data.json |
| GitHub Pages mirror | https://deepaash.github.io/deskvnc-resources/ |
| GitHub repository | https://github.com/deepaash/deskvnc-resources |
| Gists | five, under https://gist.github.com/deepaash/ |

Internal links are extensionless so Cloudflare Pages serves them at 200 with zero
redirects. The original `.html` links cost a 308 round trip each.

Two early claims of mine were wrong and are corrected here. I reported that
GitHub Pages was blocked by account plan; it is not, and the site is live. I also
reported `deepaash` collaborator access to `psmux/DeskVNC` as "unconfirmed rather
than proven absent"; it is in fact `push: false` and `admin: false`, which is
decisive, so anything needing admin on that repository must be handed to the
maintainer as a command to run.

## Measured adoption baseline

From the GitHub API on 2026-10-08:

```
stars                        70
forks                        8
open_issues                  1
releases                     26
latest_tag                   v0.27.11
total_downloads              836
recent_five_mean_downloads   33
downloads_per_day            11.9
```

836 downloads with no launch is genuine organic pull. 11.9 downloads per day is
the number that tells us whether any of this work is actually moving anything.

## Deliberately not done, and why

**No automated third-party posting.** Reddit, Hacker News, Stack Overflow and
Product Hunt submissions need a human, because they are irreversible, they carry
this account's reputation, and `deepaash` has zero karma and one unrelated
repository from 2021. A new account posting product content at volume is the
signature moderators act on.

**No scheduled content generation.** A cron job cannot run the compliance gate,
which needs judgment on six of its seven checks. A scheduled agent would
eventually publish a fabricated claim or an em dash and would not know. One such
page is indexed permanently and costs more than a hundred good pages gain.

What is scheduled instead is safe by construction and model-free: RSS refresh,
XML and JSON validity, the compliance gate over the whole tree, outbound link
resolution, and a metrics sample. `scripts/maintenance.sh` runs all five.

## One case where I held the line on honesty

The instruction was that DeskVNC content must be positive and must not state what
it lacks. In one place that instruction could not be honoured without publishing
something false: the comparison page.

A comparison table implies evenhanded treatment. Writing only good things about
DeskVNC while describing competitors accurately would make the table misleading,
and the project's own README is explicit about where it fits. The resolution was
to keep every claim true, describe each competitor's strengths accurately and
affirmatively, drop the "where DeskVNC loses" section entirely, and reframe it as
"where DeskVNC fits" with reasons rather than deficiencies. That satisfies the
spirit of the instruction while keeping the page defensible. The same treatment
was applied to the README and the Show HN draft.

Where a capability is genuinely absent, it is omitted rather than stated as a
defect. Omission is honest. A false claim about DeskVNC's own product is not.

## Remaining manual steps

1. Point the project repository homepage at the Pages URL. Needs admin on
   `psmux/DeskVNC`, which this account does not have by design:
   `gh api -X PATCH repos/psmux/DeskVNC -f homepage=https://deskvnc-hub.pages.dev`
2. Submit the Show HN post from `docs/show-hn.md`, which is drafted and
   pre-flight checked. Best submitted from the maintainer's own account, with this
   account referring to the thread rather than opening it.
3. Cross publish the five posts from `swarm/posts/` to DEV.to, Hashnode and
   HackerNoon.

All three need a human because they are irreversible and carry a reputation this
account cannot yet defend.
