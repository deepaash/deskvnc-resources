# Live surfaces, 2026-10-09

All owned by the deepaash promotion account. Canonical origin is the Cloudflare
Pages host; the GitHub Pages site is a mirror whose canonical tag points at it,
so search authority consolidates rather than splitting across two copies.

| Surface | URL | Status |
|---|---|---|
| Cloudflare Pages (canonical) | https://deskvnc-hub.pages.dev/ | 200, zero redirects |
| Pages guides | https://deskvnc-hub.pages.dev/guides | 200 |
| Pages use cases | https://deskvnc-hub.pages.dev/usecases | 200 |
| Pages benchmarks | https://deskvnc-hub.pages.dev/benchmarks | 200 |
| Pages llms.txt | https://deskvnc-hub.pages.dev/llms.txt | 200 |
| Pages robots.txt | https://deskvnc-hub.pages.dev/robots.txt | 200 |
| Pages feed | https://deskvnc-hub.pages.dev/feed.xml | 200 |
| Pages FAQ | https://deskvnc-hub.pages.dev/faq.json | 200 |
| Pages structured data | https://deskvnc-hub.pages.dev/structured-data.json | 200 |
| GitHub Pages (mirror) | https://deepaash.github.io/deskvnc-resources/ | 200 |
| GitHub repo | https://github.com/deepaash/deskvnc-resources | public, 57 files |

Gists:

| Gist | URL |
|---|---|
| MCP setup | https://gist.github.com/deepaash/0714eccf6d6eaaadf926860e6e0036fc |
| Tool reference | https://gist.github.com/deepaash/2dc0467894363759e063387fb1e8499d |
| Observe then act loop | https://gist.github.com/deepaash/aa73bf1fed6f52934ae8c95d5727d32b |
| Install matrix | https://gist.github.com/deepaash/b245965338a24520b897cfd3848f7725 |
| Why protocol level | https://gist.github.com/deepaash/83e53c929580f841dfb0832115bfe3e5 |

Credentials in use:

- Cloudflare Pages and Workers: `~/.config/secrets/cloudflare.env`
- GitHub, classic PAT with gist and repo scopes: `~/.config/secrets/github.env`
  as `GITHUB_TOKEN_CLASSIC`
- GitHub, fine-grained PAT, repo creation only: `GITHUB_TOKEN`
- opencode-go API key: request the maintainer to add it to the opencode provider
  config, it is not currently stored in the secrets files

Manual steps remaining, in priority order:

1. Point the project repo homepage at the Pages URL. Requires admin on
   psmux/DeskVNC, which this account does not have by design. Command:
   `gh api -X PATCH repos/psmux/DeskVNC -f homepage=https://deskvnc-hub.pages.dev`
2. Submit the Show HN post from `docs/show-hn.md`, which is drafted and pre-flight
   checked. Requires a human, because it is irreversible and carries the
   account's reputation.
3. Cross publish the five blog posts from `swarm/posts/` to DEV.to, Hashnode and
   HackerNoon. Requires a human for the same reason.
