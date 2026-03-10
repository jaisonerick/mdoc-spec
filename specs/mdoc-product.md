# mdoc — Product Specification

## Problem

Teams that write documentation, contracts, proposals, or technical content in Markdown frequently need to share polished, branded documents with stakeholders who expect Google Docs. The current workflow is painful:

1. **Manual formatting** — Copy-paste from Markdown preview into Google Docs loses structure, then spend 20+ minutes re-applying heading styles, code formatting, table borders, and fonts.
2. **Inconsistent branding** — Each team member applies slightly different fonts, sizes, and spacing. There's no single source of truth for document styles.
3. **No repeatability** — When the Markdown source changes, the entire manual process starts over. There's no way to update an existing Google Doc from updated Markdown.
4. **Code blocks are especially broken** — Google Docs has no native syntax highlighting. Teams resort to screenshots or unstyled monospace text.

Existing tools (pandoc to docx then upload, or browser extensions) produce mediocre formatting, don't support Google Docs API natively, or lack configurability for org-specific branding.

## What This Is

`mdoc` is a Go CLI tool that reads a Markdown file, parses it into an AST, converts the AST into Google Docs API `batchUpdate` requests, and creates or updates a Google Doc. It outputs the document URL. All formatting — fonts, sizes, colors, spacing, code theme — is controlled through a layered YAML config system, so teams get consistent, branded output every run.

The CLI is distributed via GitHub Releases with a curl-based installer and built-in auto-update. Authentication flows through a companion backend API (`mdoc-api`) that handles OAuth2 secrets and image hosting.

## What This Is NOT

- **Not a bidirectional sync tool.** Markdown → Google Docs only. Does not read Google Docs.
- **Not a real-time collaboration tool.** Runs as a CLI, produces a doc, exits.
- **Not a template engine.** No variable interpolation, conditionals, or loops. Converts Markdown as-is.
- **Not a sharing/permissions manager.** Creates docs in Google Drive; users manage access through Drive.
- **Not a general-purpose API.** `mdoc-api` exists solely to support the CLI — it is not a standalone service for third-party integrations.

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    mdoc CLI (Go)                     │
│  Markdown → AST → Google Docs API Requests → Doc    │
└──────────────┬──────────────────────┬───────────────┘
               │                      │
               ▼                      ▼
┌──────────────────────┐  ┌──────────────────────────┐
│   mdoc-api (Go)      │  │   Google Docs API        │
│  - OAuth2 redirect   │  │   - batchUpdate          │
│  - Image upload      │  │   - documents.create     │
│  - Token exchange    │  │   - documents.get        │
└──────────────────────┘  └──────────────────────────┘
```

**mdoc CLI** — Published on GitHub (`jaisonerick/mdoc`). Installable via `curl` one-liner. Auto-updates on new releases. Contains the Markdown parser, Google Docs API converter, config system, and all user-facing commands.

**mdoc-api** — Published on GitHub (`jaisonerick/mdoc-api`), separate repository. Serverless TypeScript service (Hono framework) deployed on AWS (Lambda + API Gateway) on NexaEdge infrastructure. Two responsibilities:
- **OAuth2 authorization** — Holds Google OAuth client secrets. Initiates the consent flow, exchanges the authorization code for tokens, and redirects tokens back to the CLI's local callback server.
- **Ephemeral image hosting** — Receives image uploads from the CLI, stores them in S3 with a public URL, and automatically deletes them after 10 minutes. Images only need to live long enough for Google Docs to fetch and embed them.

The CLI never holds OAuth client secrets directly — it initiates auth via the API, receives tokens back, and stores them locally.

## Data Model

### Core Entities

| Entity | Description | Source |
|--------|-------------|--------|
| **Markdown File** | Input `.md` file with optional YAML frontmatter | Filesystem |
| **Frontmatter** | YAML header controlling per-document settings (title, doc_id, folder, style overrides). Unknown fields are preserved but ignored. | Parsed from Markdown file |
| **AST** | Abstract syntax tree of the Markdown content (goldmark) | Parsed from Markdown body |
| **Style Config** | Layered YAML config defining all formatting rules | Config files (built-in → home → local → frontmatter) |
| **Syntax Theme** | Color and style definitions for code syntax highlighting tokens | Chroma built-in themes or custom theme files |
| **Google Doc** | The output document created/updated via Google Docs API | Google Drive |
| **OAuth Session** | Temporary state during auth flow: session ID, redirect URI, PKCE verifier | mdoc-api (in-memory or short-lived store) |
| **Uploaded Image** | Temporary image stored in S3 with public URL, auto-deleted after 10 minutes | mdoc-api → S3 |
| **Auth Token** | Google OAuth2 access + refresh token pair, stored per-account | CLI filesystem (`~/.config/mdoc/accounts/<name>/token.json`) |

### AST Node Types → Google Docs Mapping

| Markdown Element | AST Node | Google Docs Representation |
|-----------------|----------|---------------------------|
| Paragraph | `ast.Paragraph` | `InsertText` + `UpdateParagraphStyle` (normal text style) |
| Heading 1-6 | `ast.Heading` | `InsertText` + `UpdateParagraphStyle` (configurable per level) |
| Bold | `ast.Emphasis` (strong) | `UpdateTextStyle` with `bold: true` |
| Italic | `ast.Emphasis` | `UpdateTextStyle` with `italic: true` |
| Strikethrough | `ast.Strikethrough` | `UpdateTextStyle` with `strikethrough: true` |
| Inline code | `ast.CodeSpan` | `UpdateTextStyle` with monospace font (inherits surrounding font size, no background). If language-detectable, apply syntax theme colors; otherwise use configurable default color (default: bright red `#cc0000`, overridable in config/frontmatter). |
| Code block | `ast.FencedCodeBlock` | `InsertTable` (1×1) + cell background via `UpdateTableCellStyle` + syntax-highlighted text runs via chroma tokenizer + `UpdateTextStyle` per token |
| Unordered list | `ast.List` | `InsertText` + `CreateParagraphBullets` (BULLET_DISC_CIRCLE_SQUARE) |
| Ordered list | `ast.List` (ordered) | `InsertText` + `CreateParagraphBullets` (NUMBERED_DECIMAL_ALPHA_ROMAN) |
| Nested lists | `ast.ListItem` depth | Indentation via `nestingLevel` in bullet preset |
| Blockquote | `ast.Blockquote` | `UpdateParagraphStyle` with left indent + paragraph border-left |
| Table | `ast.Table` | `InsertTable` + smart column widths + `UpdateTableCellStyle` for borders/padding/vertical alignment (top) |
| Link | `ast.Link` | `UpdateTextStyle` with `link.url` |
| Image | `ast.Image` | Upload to mdoc-api → get public URL → `InsertInlineImage` |
| Horizontal rule | `ast.ThematicBreak` | **Discarded** — not rendered |
| Line break | `ast.Hardbreak` | `InsertText` with newline |
| Footnote | `ast.Footnote` | `CreateFootnote` + `InsertText` inside footnote |
| Mermaid/extensions | Fenced code with special lang | Rendered as regular code block (future: diagram rendering) |

### Table Column Width Strategy

Tables use smart column width calculation:

1. **Analyze cell content across all rows** for each column.
2. **Classify each column:**
   - **Fit-to-content** — All cells are short (numbers, single words, short phrases, dates). Column width is set to the minimum needed.
   - **Proportional** — Cells contain longer text. Width is distributed proportionally among proportional columns, using remaining space after fit-to-content columns.
3. **Respect Markdown alignment** — Column alignment syntax (`:---`, `:---:`, `---:`) maps to left/center/right horizontal alignment.
4. **Vertical alignment** — Default `TOP` for all cells.
5. **Override via config** — Users can set default column width strategy in config (e.g., all-equal, all-proportional, or smart/auto).

## Pipeline

```
┌──────────────┐    ┌───────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│ 1. Load      │───►│ 2. Parse  │───►│ 3. Resolve   │───►│ 4. Convert    │───►│ 5. Execute   │
│    Config    │    │  Markdown │    │    Styles    │    │   AST→Ops    │    │   API Calls  │
└──────────────┘    └───────────┘    └──────────────┘    └───────────────┘    └──────────────┘
                                                                                     │
                                                                              ┌──────▼──────┐
                                                                              │ 6. Output   │
                                                                              │    URL +    │
                                                                              │    Upsert   │
                                                                              └─────────────┘
```

### Phase 1 — Load Config

**Input:** Built-in defaults, `~/.config/mdoc/config.yaml`, `./mdoc.yaml`.

**Process:**
1. Load embedded default config (compiled into binary). This config produces a complete, polished document with no additional setup.
2. If `~/.config/mdoc/config.yaml` exists, deep-merge it over defaults.
3. If `./mdoc.yaml` exists in the current directory, deep-merge it over the result.
4. No CLI flag overrides for style, theme, or folder — all formatting comes from config/frontmatter.

**Output:** Resolved `StyleConfig` struct.

**Why layered:** Teams distribute a shared config in their repo (`./mdoc.yaml`) for consistent branding. Individuals customize at home level. The built-in defaults ensure it works out of the box with zero configuration.

### Phase 2 — Parse Markdown

**Input:** Markdown file path from CLI argument.

**Process:**
1. Read the file.
2. Extract YAML frontmatter (if present). Preserve all fields, including unknown ones.
3. Parse remaining Markdown body into AST using `goldmark` with extensions (tables, strikethrough, footnotes).
4. Merge frontmatter `styles` overrides into the resolved config from Phase 1.
5. Extract metadata: `title` (frontmatter > first H1 > filename), `doc_id`, `folder`.

**Output:** Parsed AST + final merged config + frontmatter metadata.

### Phase 3 — Resolve Styles

**Input:** Final merged config, syntax theme name.

**Process:**
1. Resolve all style definitions into concrete Google Docs API style objects (fonts, sizes as PT, colors as RGB float values, spacing in PT).
2. Load syntax highlighting theme from `chroma` by name (e.g., `monokai`, `dracula`, `github`) or from custom theme file path.
3. Pre-compute token styles: for each chroma token type (keyword, string, comment, operator, etc.), resolve to Google Docs `TextStyle` (foreground color, bold, italic).

**Output:** `ResolvedStyles` struct with ready-to-use API values.

### Phase 4 — Convert AST to Operations

**Input:** AST, `ResolvedStyles`.

**Process:**
1. Walk the AST depth-first.
2. For each node, generate a sequence of Google Docs API `Request` objects.
3. Track the document insertion index precisely — Google Docs API requires explicit character indices for all operations.
4. **Code blocks:**
   a. Generate `InsertTable` (1×1) request.
   b. Tokenize code using `chroma` with the language specified in the fenced code block.
   c. For each token, generate `InsertText` + `UpdateTextStyle` with resolved theme colors.
   d. Set cell background via `UpdateTableCellStyle`.
   e. Batch formatting requests efficiently — merge adjacent tokens with identical styles into single text runs to minimize API requests.
5. **Inline code:**
   a. Apply monospace font at the same size as surrounding text (no explicit font-size override).
   b. No background color.
   c. If the inline code can be language-detected, apply syntax theme colors. Otherwise, render in configurable default color (default: `#cc0000`, set via `styles.code_inline.color` in config or frontmatter).
6. **Images:**
   a. For all images (local files and URLs): upload to mdoc-api, receive a public URL back.
   b. Generate `InsertInlineImage` with the public URL. Images are embedded in the doc, not referenced externally.
7. **Tables:** Apply smart column width strategy (see Data Model section).
8. **Footnotes:** Convert `[^1]` syntax to Google Docs native footnotes via `CreateFootnote`.

**Output:** Ordered list of `docs.Request` objects ready for `batchUpdate`.

**Why index tracking matters:** The Google Docs API is index-based. Inserting "Hello" at index 1 means the next insertion starts at index 6. Every operation must account for all prior insertions. A single off-by-one error corrupts the entire document.

### Phase 5 — Execute API Calls

**Input:** List of `docs.Request` objects, frontmatter metadata, resolved config.

**Process:**
1. Authenticate: if no token exists, initiate OAuth2 flow via mdoc-api (open browser → user logs in → API exchanges code → CLI receives and stores token).
2. **Upsert logic:**
   a. If frontmatter contains `doc_id`: clear existing doc content (delete range index 1 to end), then apply new requests via `batchUpdate`.
   b. If no `doc_id` or doc not found: create new doc via `documents.create`. If `folder` is specified (frontmatter or config), move doc to that Drive folder (auto-create folder path if needed). Write `doc_id` back to Markdown frontmatter.
   c. **Destructive update confirmation:** When updating an existing doc (upsert with `doc_id`), fetch the existing document via `documents.get` to retrieve its current title, then prompt: `"Document 'Q1 Engineering Report' (1aBcDe...XyZ) will be overwritten. Continue? [y/N]"`. The title comes from the Google Doc itself (not frontmatter), so the user can verify they're updating the right document. Skip prompt if `--yes` flag is provided.
3. **Rate limiting:** Enforce proactive RPS limit based on Google Docs API quotas as first line of defense. Apply exponential backoff with jitter on 429/5xx responses as fallback.
4. **Batching:** Split large request lists into chunks respecting `batchUpdate` size limits. Maintain index correctness across batches.

**Output:** Google Doc ID and URL.

### Phase 6 — Output & Upsert

**Input:** Google Doc ID, original Markdown file.

**Process:**
1. Print the Google Docs URL to stdout: `https://docs.google.com/document/d/{doc_id}/edit`
2. If `doc_id` was newly created, write it back to the Markdown file's frontmatter, preserving all existing fields and content exactly.
3. Exit 0 on success.

**Output:** URL on stdout, exit code 0.

## CLI Interface

```
mdoc — Push Markdown to Google Docs

USAGE:
    mdoc push <file.md>          Create or update a Google Doc from Markdown
    mdoc auth [--account NAME]   Authenticate with Google (OAuth2 flow)
    mdoc accounts                List configured accounts
    mdoc config                  Show resolved config (merged layers)
    mdoc styles                  Show all available style properties and their defaults
    mdoc frontmatter             Show all supported frontmatter fields with descriptions
    mdoc themes                  List available syntax highlighting themes
    mdoc version                 Print version and check for updates
    mdoc update                  Update to the latest version

FLAGS (for `push`):
    --account NAME     Use a specific authenticated account (default: default account)
    --yes, -y          Skip confirmation prompt on upsert (destructive update)
    --verbose          Show detailed processing information
    --debug            Show API request/response details
    --strict           Fail on unsupported Markdown elements instead of graceful degradation
    --dry-run          Parse and convert but don't call Google APIs. Show request summary.

EXAMPLES:
    mdoc auth                          # First-time setup
    mdoc push report.md                # Create/update doc, print URL
    mdoc push report.md --verbose      # See what's happening
    mdoc push report.md --dry-run      # Preview without creating
    mdoc styles                        # See all style options
```

### Multi-Account Support

Users can authenticate multiple Google accounts:

```
mdoc auth                        # Authenticate default account
mdoc auth --account work         # Authenticate "work" account
mdoc push report.md --account work
```

Tokens are stored per-account at `~/.config/mdoc/accounts/<name>/token.json`. Initially file-based; future versions may support macOS Keychain.

## File Formats

### Markdown Input with Frontmatter

```yaml
---
title: "Q1 Engineering Report"
doc_id: "1aBcDeFgHiJkLmNoPqRsTuVwXyZ"  # auto-populated after first push
folder: "Engineering/Reports"
account: "work"                          # which authenticated account to use
toc: true                                # generate table of contents
styles:
  heading1:
    font_size: 28
    color: "#1a365d"
# Unknown fields are preserved but ignored by mdoc
author: "Jane Doe"
status: "draft"
---

# Q1 Engineering Report

## Summary

We shipped **42 features** and fixed _137 bugs_ this quarter.

Here is some `inline code` that renders in red, and some `fmt.Println("hello")` in Go colors.

### Code Example

```python
def deploy(env: str) -> bool:
    """Deploy to the specified environment."""
    return run_pipeline(env, strict=True)
```

| Metric | Q4 | Q1 | Delta |
|--------|----|----|-------|
| Uptime | 99.2% | 99.8% | +0.6% |
| Deploys | 89 | 134 | +50% |

> This is a blockquote with **bold** and _italic_ text.

See the full report[^1] for details.

[^1]: Available on the internal wiki.
```

### Config File (`~/.config/mdoc/config.yaml` or `./mdoc.yaml`)

```yaml
# Google Drive settings
drive:
  default_folder: ""              # Drive folder path (e.g., "Team/Docs"). Empty = Drive root.

# Document styles — all properties shown with defaults
styles:
  page:
    margin_top: 72                # points (1 inch = 72pt)
    margin_bottom: 72
    margin_left: 72
    margin_right: 72
    page_size: "LETTER"           # LETTER, A4, LEGAL

  normal:
    font_family: "Arial"
    font_size: 11                 # points
    bold: false
    italic: false
    underline: false
    strikethrough: false
    color: "#333333"              # hex RGB
    line_spacing: 1.15            # multiplier
    space_before: 0               # points
    space_after: 6                # points
    alignment: "START"            # START, CENTER, END, JUSTIFIED

  heading1:
    font_family: "Arial"
    font_size: 26
    bold: true
    italic: false
    underline: false
    color: "#1a1a1a"
    line_spacing: 1.15
    space_before: 24
    space_after: 8
    alignment: "START"

  heading2:
    font_family: "Arial"
    font_size: 20
    bold: true
    italic: false
    underline: false
    color: "#2d2d2d"
    line_spacing: 1.15
    space_before: 18
    space_after: 6
    alignment: "START"

  heading3:
    font_family: "Arial"
    font_size: 16
    bold: true
    italic: false
    underline: false
    color: "#404040"
    line_spacing: 1.15
    space_before: 14
    space_after: 4
    alignment: "START"

  heading4:
    font_family: "Arial"
    font_size: 14
    bold: true
    italic: false
    underline: false
    color: "#555555"
    line_spacing: 1.15
    space_before: 12
    space_after: 4
    alignment: "START"

  heading5:
    font_family: "Arial"
    font_size: 12
    bold: true
    italic: false
    underline: false
    color: "#666666"
    line_spacing: 1.15
    space_before: 10
    space_after: 2
    alignment: "START"

  heading6:
    font_family: "Arial"
    font_size: 11
    bold: true
    italic: true
    underline: false
    color: "#777777"
    line_spacing: 1.15
    space_before: 8
    space_after: 2
    alignment: "START"

  code_inline:
    font_family: "Roboto Mono"
    # font_size: inherited from surrounding text (not configurable)
    color: "#cc0000"              # default color for non-language-detected code (configurable)
    # language-detected code uses syntax theme colors instead

  blockquote:
    font_family: "Arial"
    font_size: 11
    bold: false
    italic: true
    underline: false
    color: "#666666"
    line_spacing: 1.15
    space_before: 6
    space_after: 6
    alignment: "START"
    indent_left: 36               # points
    border_left_color: "#cccccc"
    border_left_width: 3          # points

  link:
    color: "#1155cc"
    underline: true

  # Table configuration
  table:
    header_background: "#f5f5f5"
    header_bold: true
    border_color: "#dddddd"
    border_width: 0.5             # points
    cell_padding: 5               # points
    font_size: 10
    vertical_alignment: "TOP"     # TOP, MIDDLE, BOTTOM
    column_width_strategy: "smart" # smart, equal, proportional
    # smart: fit-to-content for short/numeric columns, proportional for text

  # List configuration
  list:
    indent_per_level: 36          # points
    space_between_items: 2        # points

# Code block settings
code:
  theme: "monokai"                # chroma theme name or path to custom theme file
  font_family: "Roboto Mono"
  font_size: 9
  cell_padding: 8                 # points inside the code table cell
  line_numbers: false
  # background_color is derived from theme; override here if needed
  background_color: ""
```

## Distribution & Auto-Update

Follows the same pattern as `plaud-cli` (`jaisonerick/plaud-cli`). Use it as the template for build, release, and update infrastructure.

### Installation

```bash
curl -fsSL https://raw.githubusercontent.com/jaisonerick/mdoc/main/install.sh | sh
```

The install script (modeled on `plaud-cli/install.sh`):
1. Detects OS (`darwin`/`linux`) and architecture (`x86_64`→`amd64`, `aarch64`/`arm64`→`arm64`).
2. Fetches latest release tag from GitHub API.
3. Downloads platform-specific `tar.gz` archive.
4. Extracts binary and installs to `/usr/local/bin/mdoc`.
5. Sets executable permissions.

### Release Flow

- **GoReleaser** (`.goreleaser.yml`) builds binaries with `CGO_ENABLED=0` for darwin/linux × amd64/arm64.
- Version injected via ldflags: `-X github.com/jaisonerick/mdoc/cmd.Version={{ .Version }}`.
- Archives named `mdoc_{{ .Os }}_{{ .Arch }}.tar.gz` + `checksums.txt`.
- **GitHub Actions** (`.github/workflows/release.yml`) triggers on `v*` tag push, runs GoReleaser with `release --clean`.

### Auto-Update

Same mechanism as plaud-cli's `cmd/update.go`:
- **Background check:** After every command (except `update`), checks GitHub API for latest release. At most once per day (cached in `~/.config/mdoc/update-state.json`).
- **Notice:** If newer version found, prints to stderr: `A new version of mdoc is available (v1.2.3). Run 'mdoc update' to upgrade.`
- **`mdoc update` command:** Downloads platform-specific archive, extracts binary, replaces executable (direct rename or `sudo cp` fallback for `/usr/local/bin`).
- Errors during check are silently ignored — never blocks the user's workflow.

## mdoc-api — Companion Backend

`mdoc-api` is a small, serverless Go service deployed on AWS Lambda + API Gateway. It lives in a separate repository (`jaisonerick/mdoc-api`) but is part of the mdoc product. It has exactly two responsibilities: OAuth2 authorization and ephemeral image hosting.

### Architecture

```
┌──────────────┐     ┌─────────────────────────────────────┐     ┌──────────────┐
│  mdoc CLI    │────►│         mdoc-api (Lambda)            │────►│   Google     │
│              │◄────│                                     │◄────│   OAuth2     │
│  localhost   │     │  API Gateway → Lambda (Go)          │     │   Server     │
│  :9876       │     │                                     │     └──────────────┘
└──────────────┘     │  ┌──────────┐  ┌─────────────────┐  │
                     │  │ /auth/*  │  │ /images/upload   │  │
                     │  └──────────┘  └────────┬────────┘  │
                     └─────────────────────────┼───────────┘
                                               │
                                          ┌────▼────┐
                                          │   S3    │
                                          │ (images)│
                                          └─────────┘
```

### OAuth2 Flow

The CLI never holds Google OAuth client secrets. The full flow:

1. User runs `mdoc auth`.
2. CLI starts a temporary HTTP server on `localhost:9876` (callback receiver).
3. CLI opens the browser to `https://<mdoc-api>/auth/start?redirect_uri=http://localhost:9876/callback`.
4. mdoc-api redirects the browser to Google's OAuth consent screen with the correct scopes (Docs, Drive), client ID, and a server-generated PKCE challenge.
5. User grants access on Google's consent screen.
6. Google redirects back to mdoc-api with the authorization code.
7. mdoc-api exchanges the code for access + refresh tokens using the client secret (stored securely in the backend, e.g., environment variable or AWS Secrets Manager).
8. mdoc-api redirects the browser to the CLI's local callback URL (`http://localhost:9876/callback`) with the tokens as query parameters (or POST body).
9. CLI captures the tokens, stores them at `~/.config/mdoc/accounts/<name>/token.json`, and shuts down the local server.
10. CLI prints "Authorization successful!" and exits.

**Token refresh:** The CLI handles token refresh directly using `golang.org/x/oauth2`. Refresh tokens work without client secrets when using the PKCE flow. If a refresh fails, the CLI prompts the user to re-authenticate via `mdoc auth`.

### Image Upload & Ephemeral Hosting

Local images in Markdown files can't be referenced directly in Google Docs — they need a publicly accessible URL. mdoc-api provides ephemeral image hosting:

1. CLI reads a local image file during `mdoc push`.
2. CLI uploads the image to `POST https://<mdoc-api>/images/upload` with multipart form data. The request includes the user's Google access token in the `Authorization` header as proof of identity.
3. mdoc-api stores the image in an S3 bucket and returns the public URL.
4. CLI uses the public URL in `InsertInlineImage` Google Docs API request.
5. Google Docs fetches the image from the URL and embeds it in the document.
6. **After 10 minutes, the image is automatically deleted from S3** (via S3 lifecycle rules). The image only needs to live long enough for Google Docs to fetch and embed it.

**Why ephemeral:** Images are embedded (copied) into the Google Doc by Google's servers during the API call. Once embedded, the source URL is no longer needed. Keeping images around would waste storage and create data retention concerns.

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/auth/start` | None | Initiates OAuth2 flow. Accepts `redirect_uri` param. Redirects to Google. |
| `GET` | `/auth/callback` | None | Receives Google's auth code. Exchanges for tokens. Redirects to CLI's local callback with tokens. |
| `POST` | `/images/upload` | Google access token | Accepts multipart image upload. Stores in S3. Returns `{"url": "https://..."}`. |
| `GET` | `/images/{id}` | None | Serves an uploaded image (public, used by Google Docs to fetch). Returns 404 after expiration. |
| `GET` | `/health` | None | Health check. Returns `{"status": "ok"}`. |

### API Authentication

The API endpoints themselves do not require an API key. The OAuth2 start endpoint is public (anyone can initiate a flow, but they still need a Google account to complete it). The image upload endpoint requires a valid Google access token in the `Authorization` header — this proves the uploader is an authenticated mdoc user.

### Infrastructure

- **Deployment:** AWS Lambda + API Gateway (serverless, pay-per-use)
- **Image storage:** S3 bucket with public-read ACL on objects
- **Image expiration:** S3 lifecycle rule deletes objects after 10 minutes
- **Secrets:** Google OAuth client ID and secret stored in Lambda environment variables or AWS Secrets Manager
- **Domain:** Configured in NexaEdge infrastructure (Route53), e.g., `api.mdoc.dev`
- **HTTPS:** Via API Gateway (automatic certificate management)
- **CI/CD:** GitHub Actions, deploy on push to main

## Core Constraints

1. **Index correctness is paramount.** Every Google Docs API request references character indices. The conversion phase must maintain a precise running index. Extensive testing with known inputs and expected indices is required.

2. **Config is always valid.** The built-in default config must produce a complete, polished document with zero configuration. Every style property has a sensible default. Missing user config values fall back to defaults — never to zero/empty.

3. **Frontmatter is sacred.** When writing `doc_id` back to the Markdown file, the tool must preserve all existing frontmatter fields (including unknown ones) and Markdown content exactly. No reformatting, no reordering, no trailing whitespace changes.

4. **Idempotent upsert.** Running the tool twice on the same file with a `doc_id` produces the same Google Doc content. The clear-and-rewrite strategy ensures no stale content remains.

5. **Graceful degradation by default.** If a Markdown element can't be perfectly represented in Google Docs, produce the best approximation and log a warning. The `--strict` flag changes this to fail with an error instead.

6. **Works out of the box.** Zero configuration needed for a good-looking document. Sensible defaults for everything: styles, theme, folder placement, column widths.

7. **No secrets in the CLI.** OAuth2 client credentials live in mdoc-api, never in the CLI binary or local config. Only user tokens are stored locally.

8. **Proactive rate limiting.** Enforce Google API RPS limits client-side before hitting quotas. Exponential backoff with jitter as fallback for 429/5xx responses.

9. **Images are ephemeral.** Uploaded images are deleted from S3 after 10 minutes. They exist only long enough for Google Docs to fetch and embed them. No long-term image storage.

10. **mdoc-api is stateless.** No database, no sessions beyond the OAuth flow. Lambda handles requests independently. All persistent state lives in S3 (images, briefly) or in the CLI (tokens).

## Success Criteria

1. `mdoc push report.md` produces a Google Doc URL on stdout with zero prior configuration.
2. All Markdown elements in the mapping table render correctly in the output doc.
3. Code blocks display syntax-highlighted text in a single table cell with themed background, using chroma tokenization.
4. Inline code renders in monospace at surrounding text size, in red or syntax-colored as appropriate.
5. Running `mdoc push` again on the same file (with `doc_id` in frontmatter) updates the existing doc.
6. The `doc_id` is written back to frontmatter on first creation, preserving all existing content.
7. Config layering works: built-in → home → local → frontmatter.
8. At least 5 chroma themes work out of the box (monokai, dracula, github, solarized-dark, one-dark).
9. Tables use smart column widths with correct vertical (top) and horizontal alignment.
10. A team distributes `mdoc.yaml` in their repo and gets consistent formatting across members.
11. `mdoc auth` completes the OAuth2 flow via mdoc-api and stores tokens locally.
12. Images (local and URL) are uploaded and embedded in the document.
13. `mdoc styles`, `mdoc frontmatter`, and `mdoc themes` provide useful reference output.
14. Installation works via `curl` one-liner on macOS and Linux.
15. Auto-update check runs non-blocking and `mdoc update` works.
16. Footnotes convert to native Google Docs footnotes.
17. Documents of at least 50 pages process without errors or timeouts.
18. `mdoc-api` health check endpoint responds successfully.
19. OAuth2 flow via mdoc-api completes end-to-end: CLI → browser → Google → mdoc-api → CLI callback → token stored.
20. Local image upload via mdoc-api returns a public URL that Google Docs can fetch.
21. Uploaded images are automatically deleted from S3 after 10 minutes.
22. mdoc-api handles concurrent requests without errors (multiple users pushing simultaneously).

## Open Questions

1. **Inline code language detection** — How should `mdoc` determine if inline code is a specific language? Options: (a) always red unless in a context like a code-heavy doc with a dominant language, (b) use heuristics from chroma's `Analyse`, (c) frontmatter field to set default inline language.
2. **Table width hints in Markdown** — Standard Markdown has no column width syntax. Should `mdoc` support a custom comment syntax (e.g., `<!-- mdoc:widths 30% 20% 50% -->`) above tables, or rely entirely on the smart auto-detection?
3. **mdoc-api domain** — What domain? `api.mdoc.dev`? Requires DNS setup in NexaEdge infrastructure (Route53).
4. **Image size/scaling** — Should images be inserted at original size, scaled to page width, or configurable per-image?
5. **Nested blockquotes** — Google Docs has limited support. Increase indent per nesting level? Use different border colors?
6. **TOC placement** — When `toc: true` is set in frontmatter, insert at the beginning of the document (after title) or at a `<!-- toc -->` marker?
7. **OAuth2 PKCE vs. server-side exchange** — Should the flow use PKCE (allowing the CLI to refresh tokens without client secrets) or a traditional server-side exchange (mdoc-api always involved in refresh)?
8. **S3 lifecycle precision** — S3 lifecycle rules have a minimum granularity of 1 day for expiration. For 10-minute TTL, we may need a Lambda-based cleanup (scheduled every few minutes) or S3 Object Expiration with a workaround. Investigate during architecture.
9. **Local callback port** — Should the CLI use a fixed port (9876) or find an available port dynamically? Fixed is simpler but may conflict.
