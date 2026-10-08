# DeskVNC keyword and intent map

This document maps the search intents a person (or an AI agent on a person's behalf) would actually have to the DeskVNC resource that should serve each one, and flags the intents that are not yet covered by anything in the swarm.

Use it to choose the right surface when writing new content, and to spot the gaps worth filling next.

## How to read this map

- **Intent**: the question or job to be done, phrased the way someone would type it or speak it.
- **Primary surface**: the DeskVNC asset that already answers this intent, or comes closest.
- **Status**: `covered` (an asset already serves this intent), `partial` (an asset touches it but not in a way a searcher or an LLM could answer with), `gap` (no asset yet, worth writing).
- **Why this matters**: the audience for the intent and the role DeskVNC plays for them.

## Core intents

### Agent driven remote desktop

- **Status**: covered
- **Primary surface**: README "Let an agent drive it" section, llms-full.txt MCP server chapter, structured-data.json `featureList`
- **Why this matters**: the highest value intent for the project. Anyone asking an AI agent to operate a real desktop lands here, and the answer is the entire reason DeskVNC exists.

### MCP remote desktop control

- **Status**: covered
- **Primary surface**: README introduction to `dvv`, llms-full.txt "What the MCP server is" chapter, faq.json entries for `dvv` and the agent loop, structured-data.json `featureList` and `keywords`
- **Why this matters**: the most precise technical phrasing for the audience that already knows what Model Context Protocol is.

### Automate VDI with AI

- **Status**: covered
- **Primary surface**: README "Citrix and VDI" framing in "Let an agent drive it", llms-full.txt "Use cases" chapter
- **Why this matters**: VDI admins are a specific audience with a specific pain: every other agent tool installs on the desktop and gets refused. The protocol approach is the answer.

### Reach Citrix desktops with an agent

- **Status**: covered
- **Primary surface**: README quote on funded tools, CONTEXT.md verified framing, llms-full.txt
- **Why this matters**: a long tail of admins search this exact phrasing when their existing tooling fails on Citrix. The DeskVNC quote from the README lands the answer directly.

### VNC RDP SSH in one client

- **Status**: covered
- **Primary surface**: README opening, faq.json protocol question, structured-data.json `featureList`
- **Why this matters**: this is the everyday "I have too many tools" pain, the audience that arrives looking for a connection manager and discovers the MCP server later.

### Remote desktop human takeover

- **Status**: covered
- **Primary surface**: README "A person can take the wheel at any moment", llms-full.txt "Why this is different" chapter, faq.json "Can a human take control back" question
- **Why this matters**: trust signal for any audience that has been burned by automation that strands a desktop mid drag. This is also a frequent AI search phrasing.

### Agent control remote machine safely

- **Status**: covered
- **Primary surface**: README four rules (attach before you act, coordinate fencing, typing fencing, human takeover), llms-full.txt "Why this is different" chapter, faq.json coordinate fencing question
- **Why this matters**: "safely" is the word a risk-averse ops or security audience uses. Fencing rules and OS keychain storage are the answer.

### Jump host automation

- **Status**: covered
- **Primary surface**: README "jump hosts" mention, llms-full.txt "Use cases" chapter
- **Why this matters**: jump host operators are a small, focused audience who already speak VNC or SSH and want their existing fabric to be agent reachable.

### Protocol level remote access

- **Status**: covered
- **Primary surface**: README "What is under it" chapter on the three protocol cores, llms-full.txt "Protocol features" chapter
- **Why this matters**: a phrasing used by engineers who care about how the wire works, not which brand owns it. The pure Rust cores are the answer.

## Adjacent intents that already land

### Native remote desktop client Windows macOS Linux

- **Status**: covered
- **Primary surface**: README, structured-data.json `operatingSystem`, faq.json platforms question

### Free open source remote desktop client

- **Status**: covered
- **Primary surface**: README "free and open source and stays that way", faq.json license question, structured-data.json `offers`

### Remote desktop with credentials in OS keychain

- **Status**: covered
- **Primary surface**: README architecture section, faq.json credentials question

### Bidirectional SFTP file transfer

- **Status**: covered
- **Primary surface**: README "In a session" section, structured-data.json `featureList`

### Tabbed multi-protocol remote desktop

- **Status**: covered
- **Primary surface**: README "Several machines at once" section, structured-data.json `featureList`

### Rust Tauri remote desktop

- **Status**: covered
- **Primary surface**: README "Building from source" section, structured-data.json `runtimePlatform`

## Intents that are partial today

### AI agent integrate with remote desktop

- **Status**: partial
- **Primary surface**: docs/AGENTS.md, llms-full.txt "Verified AI clients" section
- **Gap**: a copy paste ready tutorial per major client (Claude Code, OpenCode, Cursor, Codex, Gemini CLI, VS Code) that someone can follow without leaving the page. The integration notes are the source of truth, but a one page version per client would land the searcher faster.

### MCP tools list for remote desktop

- **Status**: partial
- **Primary surface**: llms-full.txt "MCP tools" section
- **Gap**: a dedicated reference page that lists every `dvv_*` tool with its full parameter schema. LLM answer engines tend to quote that page directly when asked "what tools does the DeskVNC MCP server expose".

### dvv_open / dvv_screen / dvv_click reference

- **Status**: partial
- **Primary surface**: README code block, llms-full.txt four call loop
- **Gap**: a single page that walks the four call loop with one worked example per protocol (VNC, RDP, SSH) so an LLM that has only seen the snippet can answer follow up questions.

## Gaps worth filling next

These are the intents a person or an LLM would plausibly have, that no current asset answers well. They are the highest leverage targets for the next round of content.

- **"Open source TeamViewer alternative"**: a high volume commercial intent with a clear, true DeskVNC story (one client for VNC, RDP and SSH, plus the agent plane). Worth a page on the official site or a long form post.
- **"MCP server for remote desktop"**: a category intent. A category explainer that positions `dvv` as the protocol level option (versus remote install tools) would serve both human search and AI search.
- **"AI agent for Windows desktop"**: a high volume intent. A focused guide on reaching Windows desktops over RDP with `dvv` would convert well, especially when paired with the "no install on the remote" framing.
- **"Remote desktop without installing on remote machine"**: a precise phrasing the VDI and locked down desktop audience uses. A page that turns the README quote into a fuller argument would land this intent cleanly.
- **"Free remote desktop for developers"**: a developer audience intent. A page that frames DeskVNC as the developer's connection manager, with the MCP server as a power feature, fits the audience.
- **"Citrix automation tool"**: a high value intent for admins. A page that walks the Citrix use case end to end would be the right home.
- **"Human in the loop remote automation"**: a phrasing used by people thinking about safety. A page that frames the takeover story and the fencing rules together is the answer.
- **"MCP tool benchmark"** or **"agent action latency"**: a technical audience intent. A page that quotes the verified benchmark table and adds the per protocol context would answer the comparison questions LLMs get.
- **"Cross-platform remote desktop client"**: a platform intent. A page that pairs the three OS targets with the three protocols and the one library model would close this loop.
- **"Wake on LAN GUI"** and **"mDNS remote desktop discovery"**: a network admin intent. The Wake on LAN and network discovery features in the README are the answer; a page that frames them as a discover and wake workflow would be the right home.
- **"Tauri remote desktop"** and **"Rust VNC client"**: a developer curiosity intent. A page that frames the pure Rust protocol cores and the Tauri 2 shell would land this audience.
- **"Vibe coding remote desktop"**: a phrasing the current generation of agent native developers uses. A page that shows `dvv` in a loop with a coding assistant would be a high fit piece of content.
- **"dvv doctor"** and **`dvv` CLI reference**: a power user intent. A reference page for the `dvv` command and its subcommands would answer the "how do I register this with my client" question in one place.

## How to use this map when writing

- Start every new piece of content with an intent from this list, not with a topic. If the intent is not on the list, decide whether it is worth adding to the list first.
- Link the new content from the entry it serves, so the map stays a live index of what we have.
- When an intent is served by more than one surface, prefer the deepest one (full doc) for the primary link, and use the llms assets for AI discovery and the FAQ for the short answer.
