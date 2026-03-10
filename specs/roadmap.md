# mdoc — Roadmap

## Vision

mdoc is a Go CLI that converts Markdown files into polished, branded Google Docs. It's for developers and teams who write in Markdown but need to share professional documents with stakeholders who expect Google Docs. One command, zero manual formatting.

## Version Progression

```
V0.1 Basic Push
 │  Paragraphs, headings, bold, italic, strikethrough, links
 │  Manual token auth, hardcoded styles, rate limiting
 │
V0.2 Full Markdown
 │  + Lists, code blocks (syntax highlighted), inline code
 │  + Tables (smart widths), blockquotes
 │  + Theme selection (monokai, dracula, github, etc.)
 │
V0.3 Config & Upsert
 │  + Layered YAML config (built-in → home → local → frontmatter)
 │  + Frontmatter parsing (title, doc_id, folder, styles)
 │  + Upsert: update existing docs, doc_id writeback
 │
V0.4 Images & Footnotes
 │  + URL-based images embedded in doc
 │  + Native Google Docs footnotes
 │  + Drive folder placement
 │
V0.5 mdoc-api (separate repo: jaisonerick/mdoc-api)
 │  + OAuth2 authorization backend (holds client secrets)
 │  + Token exchange endpoint
 │  + Image upload + public hosting endpoint
 │  + Deployed on NexaEdge infrastructure
 │
V0.6 OAuth & Image Upload
 │  + CLI integrates with mdoc-api for OAuth2 (no more manual tokens)
 │  + Multi-account support
 │  + Local image upload via mdoc-api
 │
V1.0 Distribution  ← First public release
    + curl installer, GoReleaser, GitHub Releases
    + Auto-update check + mdoc update
```

## Milestones

| Milestone | Version | Significance |
|-----------|---------|-------------|
| First doc created | V0.1 | Core pipeline works end-to-end |
| Any Markdown file works | V0.2 | Formatter is complete |
| Edit-push-edit-push workflow | V0.3 | Tool is practical for daily use |
| Rich documents | V0.4 | Images and footnotes complete the output |
| Backend service live | V0.5 | mdoc-api deployed, enabling OAuth and image upload |
| Zero-friction auth | V0.6 | No manual token management |
| **Public release** | **V1.0** | **Installable, updatable, ready for teams** |

## V1.0 Release Criteria

- All Markdown elements from the spec render correctly in Google Docs
- Layered config system with frontmatter overrides works
- Upsert (create or update) workflow is seamless
- mdoc-api backend deployed and operational
- OAuth2 authentication via mdoc-api (no manual tokens needed)
- Images (URL and local) embedded in documents
- Footnotes render as native Google Docs footnotes
- curl installer works on macOS and Linux (amd64 + arm64)
- Auto-update mechanism functional
- Rate limiting protects against API quota exhaustion
- A 50-page document processes without errors

## Post-V1.0 Direction

- **Reference commands** — `mdoc styles`, `mdoc frontmatter`, `mdoc themes`, `mdoc config` for discoverability
- **Dry-run mode** — `--dry-run` to preview without calling Google APIs
- **Strict mode** — `--strict` to fail on unsupported elements instead of graceful degradation
- **Table of Contents** — `toc: true` in frontmatter generates a TOC
- **Image scaling** — Control image size per-image or via config
- **macOS Keychain** — Secure token storage
- **Homebrew tap** — `brew install mdoc`
- **Inline code language detection** — Syntax-colored inline code via heuristics or config

## Spec Review Notes

- The product spec's image handling section assumes mdoc-api is available from V0.1. The roadmap defers this: V0.4 supports URL-only images, V0.5 builds mdoc-api, V0.6 integrates local uploads.
- V0.5 (mdoc-api) lives in a separate repository (`jaisonerick/mdoc-api`). The product spec is now part of the main mdoc spec (`specs/mdoc-product.md`, "mdoc-api — Companion Backend" section). Architecture for the API should be covered when running `/architect-version` for V0.5.
- Open question from spec: inline code language detection is deferred to post-V1.0.
