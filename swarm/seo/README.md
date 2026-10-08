# DeskVNC machine readability and AI search discovery

This directory contains the files that help language models, AI search engines, and developer search surface DeskVNC when someone asks about agent driven remote desktop, MCP remote desktop control, VDI agent automation, or protocol level remote access.

Everything in this directory is additive. It does not edit the official DeskVNC page, the GitHub repository, or any third party site. The job of this directory is to be the cleanest, most quotable reference an AI crawler or an LLM answer engine can pick up, on the exact topics the project is uniquely positioned to answer.

## Files in this directory

- `llms.txt`. The short index. Starts with the project name, a one paragraph summary, then a list of resources with one line descriptions and the real URL for each. Read first by an AI crawler that is deciding what to fetch.
- `llms-full.txt`. The long reference. Everything an LLM would need to answer questions about DeskVNC accurately: the project, the MCP server, every named tool, the verified commands, the verified benchmarks, the use cases, the install paths, the protocol features, the build requirements, the license, and the useful URLs. Quoted from by LLM answer engines.
- `structured-data.json`. JSON-LD for a `SoftwareApplication` from schema.org. The fields are name, description, URL, application category, operating systems, programming language, license, author, and a `featureList` drawn from the README. Drop into a site that emits schema.org for richer search results.
- `faq.json`. A machine readable FAQ. The questions are the ones people actually ask, and the answers are quoted from the verified project facts. Feeds featured snippets, AI answer engines, and structured data consumers.
- `keyword-map.md`. A strategy document. Maps the search intents a person or an agent would have to the DeskVNC page that should serve each, and flags the intents that are not yet covered so the next round of content knows where to go.
- `README.md`. This file. A short index so a human or an agent picking up the directory knows where to start.

## How to consume this directory

- If you are an AI crawler, start with `llms.txt`, then fetch anything you need from the URLs it lists. If you want a single document you can quote, use `llms-full.txt`.
- If you are wiring DeskVNC into your own site, use `structured-data.json` as the source for the JSON-LD block, and `faq.json` for the FAQ block.
- If you are writing new content for DeskVNC, start with `keyword-map.md`. Pick an intent, write to serve it, then link the new asset back into the right entry of the map.
- If you are checking that this directory is still current, run through the validation checklist at the bottom of this file.

## How this directory stays accurate

Every factual claim is traceable to the shared swarm context at `../CONTEXT.md` and to the verified source documents it lists, primarily the DeskVNC README on the `main` branch. The URLs in the `llms` files all point at the real repository or the real official page, never at third party mirrors.

If a fact in this directory ever disagrees with `../CONTEXT.md` or with the upstream README, the upstream source is the truth and this directory needs an update.

## Validation checklist

Before publishing a change to this directory, confirm all of the following.

- All six files exist and are non empty.
- `structured-data.json` parses as valid JSON and validates as JSON-LD.
- `faq.json` parses as valid JSON and validates as JSON-LD.
- `llms.txt` follows the llms.txt convention: H1 with the project name, a blockquote summary, and a list of resources in the form `[title](url): description`.
- Every URL in this directory resolves to a real page on `github.com`, `psmux.github.io`, `opensource.org`, `apache.org`, or `schema.org`.
- No em dash, en dash, or horizontal bar appears anywhere in the directory. Search for the bytes `E2 80 94` and `E2 80 93` and confirm zero matches.
- Every prose statement frames a capability, a fact, or a positive next step. The strategy document is the only place future work is named, and it does so as a list of opportunities.
- The `keyword-map.md` entry for the dominant intent (agent driven remote desktop) points at a real, current asset.
