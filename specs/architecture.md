# mdoc — Technical Architecture

## Technology Stack

| Layer | Technology | Latest | Preferred (Context7) | Rationale |
|-------|-----------|--------|---------------------|-----------|
| Language | Go | 1.26.1 | **1.26.0** | Latest stable (March 2026). CGO_ENABLED=0 for cross-platform. |
| CLI Framework | spf13/cobra | v1.10.2 | **v1.9.1** | Industry standard for Go CLIs. Subcommand routing, flag parsing, help generation. Same as plaud-cli. |
| Config | spf13/viper | v1.21.0 | **v1.20.1** | Built-in YAML support, layered config merging, pairs naturally with Cobra. |
| Markdown Parser | yuin/goldmark | v1.7.16 | v1.7.16 | CommonMark-compliant, extensible AST, built-in extensions for tables, strikethrough, footnotes. Pure Go. |
| Frontmatter | Manual (yaml.v3 Node) | — | — | Manual `---` splitting + yaml.v3 Node parsing. Enables lossless frontmatter writeback preserving field order, comments, and unknown fields. |
| Syntax Highlighting | alecthomas/chroma/v2 | v2.23.1 | v2.23.1 | Same highlighter used by Hugo. 200+ languages, 40+ themes. Token iterator API for per-token Google Docs styling. |
| Google Docs API | google.golang.org/api/docs/v1 | v0.270.0 | v0.270.0 | Official Go client. Type-safe request builders, handles auth token refresh. |
| Google Drive API | google.golang.org/api/drive/v3 | v0.270.0 | v0.270.0 | Same module as Docs API. Needed for folder creation, file moves. |
| OAuth2 | golang.org/x/oauth2 | v0.36.0 | v0.36.0 | Standard Google OAuth2 flow. CLI receives tokens from mdoc-api, stores and refreshes locally. |
| YAML | gopkg.in/yaml.v3 | v3.0.1 | v3.0.1 | Frontmatter preservation via yaml.Node (preserves field order, comments, formatting on writeback). |
| Rate Limiting | golang.org/x/time | v0.15.0 | v0.15.0 | Token bucket rate.Limiter for proactive Google API RPS limiting. |
| HTTP Client | net/http (stdlib) | — | — | For mdoc-api communication (auth flow, image upload). No external HTTP library needed. |
| Testing | testing (stdlib) + golden files | — | — | Golden file pattern: known Markdown input → expected API request JSON. Mock Google API client interface. |
| Build/Release | goreleaser/goreleaser | v2 | v2 | Cross-compilation, GitHub Releases, checksums. Same pattern as plaud-cli. |
| CI/CD | GitHub Actions | — | — | Release on tag push. Same workflow as plaud-cli. |

### mdoc-api Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Language | TypeScript (Node.js 22) | Fast Lambda cold starts, excellent AWS SDK support, minimal boilerplate for a small service. |
| Web Framework | Hono | Lightweight, runs natively on Lambda. Built-in middleware, routing, and request parsing. Best DX for Lambda APIs. |
| Lambda Adapter | @hono/aws-lambda | Official Hono adapter for AWS Lambda + API Gateway. Zero-config integration. |
| AWS SDK | @aws-sdk/client-s3 | S3 operations (PutObject, DeleteObject). AWS SDK v3 — tree-shakeable, modular. |
| OAuth2 | googleapis (google-auth-library) | Google OAuth2 token exchange. Official Google auth library for Node.js. |
| Runtime | AWS Lambda (Node.js 22.x) | Serverless, pay-per-use. Fast cold starts with Node.js. |
| API Gateway | AWS API Gateway v2 (HTTP API) | Lower cost and latency than REST API. Sufficient for simple routing to Lambda. |
| Image Storage | AWS S3 | Public-read objects for image hosting. 1-day lifecycle rule for automatic cleanup. |
| Infrastructure | Terraform (in NexaEdge infra repo) | Consistent with existing infrastructure management. Lambda, API Gateway, S3, Route53. |
| CI/CD | GitHub Actions | Build TypeScript, deploy to Lambda on push to main. |

### Dependency Notes

**Why manual frontmatter parsing:**
We need `gopkg.in/yaml.v3` with `yaml.Node` for lossless writeback — preserving field order, comments, and formatting when writing `doc_id` back to frontmatter. Manual `---` delimiter splitting + yaml.v3 Node parsing handles both reading and writing cleanly in a single dependency.

**Transitive dependencies from google.golang.org/api:**
- `golang.org/x/oauth2` v0.35.0+ (compatible with our v0.36.0)
- `golang.org/x/net` v0.51.0+, `golang.org/x/sync` v0.19.0+
- `cloud.google.com/go/auth` v0.18.2+
- OpenTelemetry instrumentation packages (grpc + http)

**YAML import path coexistence:**
Viper uses `go.yaml.in/yaml/v3` (new import path), we use `gopkg.in/yaml.v3` (classic). Both are the same library and coexist in Go without conflict.

**Shared sub-dependency:**
Both Cobra and Viper depend on `spf13/pflag`. Viper v1.20.1 requires pflag v1.0.10, compatible with Cobra v1.9.1.

**Future note:** Go 1.27 will drop macOS 12 Monterey support (requires macOS 13 Ventura+).

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          mdoc CLI (Go binary)                        │
│                                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Config   │─►│  Parser  │─►│  Style   │─►│ Converter│─►│Executor│ │
│  │  Loader   │  │(goldmark)│  │ Resolver │  │ (AST→Ops)│  │ (API)  │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│       │              │                            │            │  │   │
│       ▼              ▼                            ▼            │  │   │
│  ┌──────────┐  ┌──────────┐                ┌──────────┐       │  │   │
│  │  Viper   │  │ yaml.v3  │                │  chroma  │       │  │   │
│  │ (merge)  │  │  (Node)  │                │(tokenize)│       │  │   │
│  └──────────┘  └──────────┘                └──────────┘       │  │   │
│                                                                │  │   │
└────────────────────────────────────────────────────────────────┼──┼───┘
                                                                 │  │
                    ┌────────────────────────────────────────┐  │  │
                    │   mdoc-api (TypeScript + Hono)          │  │  │
                    │   repo: jaisonerick/mdoc-api            │◄─┘  │
                    │                                        │     │
                    │   API Gateway v2 (HTTP)                 │     │
                    │    └─► Lambda (Node.js 22)              │     │
                    │         ├─ GET  /auth/start → Google    │     │
                    │         ├─ GET  /auth/callback → tokens │     │
                    │         ├─ POST /images/upload → S3 URL │     │
                    │         ├─ GET  /images/:id → S3 proxy  │     │
                    │         └─ GET  /health                 │     │
                    │                     │                    │     │
                    │              ┌──────▼──────┐            │     │
                    │              │     S3      │            │     │
                    │              │  (images,   │            │     │
                    │              │  1-day TTL) │            │     │
                    │              └─────────────┘            │     │
                    └────────────────────────────────────────┘     │
                                                                    │
                    ┌────────────────────────────────┐              │
                    │     Google APIs                 │◄─────────────┘
                    │                                 │  batchUpdate
                    │  docs.googleapis.com/v1         │  documents.create
                    │  www.googleapis.com/drive/v3    │  files.create
                    └────────────────────────────────┘
```

## Project Structure

```
github.com/jaisonerick/mdoc/
├── main.go                          # Entry point, calls cmd.Execute()
├── go.mod
├── go.sum
├── .goreleaser.yml                  # GoReleaser config (from plaud-cli template)
├── .github/
│   └── workflows/
│       └── release.yml              # GitHub Actions: build + release on v* tag
├── install.sh                       # curl installer (from plaud-cli template)
│
├── cmd/                             # Cobra commands (one file per command)
│   ├── root.go                      # Root command, Version var, update check hook
│   ├── push.go                      # `mdoc push <file.md>` — main pipeline
│   ├── auth.go                      # `mdoc auth` — OAuth2 flow via mdoc-api
│   ├── accounts.go                  # `mdoc accounts` — list authenticated accounts
│   ├── config_cmd.go                # `mdoc config` — show resolved config
│   ├── styles.go                    # `mdoc styles` — show style properties + defaults
│   ├── frontmatter_cmd.go           # `mdoc frontmatter` — show supported fields
│   ├── themes.go                    # `mdoc themes` — list chroma themes
│   ├── version.go                   # `mdoc version` — print version
│   └── update.go                    # `mdoc update` — self-update (from plaud-cli)
│
├── internal/
│   ├── config/                      # Config loading and merging
│   │   ├── config.go                # StyleConfig struct, Load(), deep-merge logic
│   │   ├── defaults.go              # Embedded default config (go:embed)
│   │   └── defaults.yaml            # Default YAML config file (embedded at build)
│   │
│   ├── parser/                      # Markdown parsing + frontmatter extraction
│   │   ├── parser.go                # Parse() → AST + Frontmatter + merged config
│   │   └── frontmatter.go           # Frontmatter struct, read/write with preservation
│   │
│   ├── styles/                      # Style resolution
│   │   ├── resolver.go              # Resolve config → Google Docs API style objects
│   │   └── theme.go                 # Load chroma theme, pre-compute token styles
│   │
│   ├── converter/                   # AST → Google Docs API requests (core logic)
│   │   ├── converter.go             # Convert(ast, styles) → []docs.Request
│   │   ├── cursor.go                # Cursor struct — index tracking state machine
│   │   ├── paragraph.go             # Paragraph + heading handlers
│   │   ├── inline.go                # Bold, italic, strikethrough, inline code, links
│   │   ├── code.go                  # Fenced code block → table + chroma tokens
│   │   ├── list.go                  # Ordered/unordered/nested lists
│   │   ├── table.go                 # Table + smart column widths
│   │   ├── blockquote.go            # Blockquote with border-left
│   │   ├── image.go                 # Image upload + InsertInlineImage
│   │   ├── footnote.go              # Footnote → CreateFootnote
│   │   └── toc.go                   # Table of contents generation
│   │
│   ├── gdocs/                       # Google Docs/Drive API interaction
│   │   ├── client.go                # DocsClient interface + real implementation
│   │   ├── batch.go                 # Request batching (split large request lists)
│   │   ├── ratelimit.go             # Proactive RPS limiter + exponential backoff
│   │   └── drive.go                 # Drive operations (folder create, file move)
│   │
│   ├── auth/                        # OAuth2 authentication via mdoc-api
│   │   ├── auth.go                  # Auth flow: open browser → mdoc-api → receive token
│   │   ├── token.go                 # Token storage/retrieval per account
│   │   └── accounts.go              # Multi-account management
│   │
│   ├── images/                      # Image handling
│   │   └── upload.go                # Upload to mdoc-api, return public URL
│   │
│   └── updater/                     # Self-update mechanism (from plaud-cli)
│       └── update.go                # CheckForUpdate(), Update(), state caching
│
├── testdata/                        # Golden test files
│   ├── basic_paragraph/
│   │   ├── input.md                 # Test input
│   │   └── expected.json            # Expected API request list
│   ├── code_block_monokai/
│   │   ├── input.md
│   │   └── expected.json
│   ├── full_document/
│   │   ├── input.md
│   │   └── expected.json
│   └── ...                          # One directory per test case
│
└── specs/                           # Product and architecture specs
    ├── mdoc-product.md
    └── architecture.md
```

### mdoc-api Project Structure

```
github.com/jaisonerick/mdoc-api/
├── src/
│   ├── index.ts                     # Hono app + Lambda handler export
│   ├── routes/
│   │   ├── auth.ts                  # GET /auth/start, GET /auth/callback
│   │   ├── images.ts                # POST /images/upload, GET /images/:id
│   │   └── health.ts                # GET /health
│   ├── services/
│   │   ├── google-oauth.ts          # Google OAuth2 token exchange logic
│   │   └── s3.ts                    # S3 upload, presigned URL, object serving
│   └── config.ts                    # Environment variables, constants
├── package.json
├── tsconfig.json
├── vitest.config.ts                 # Test configuration
├── test/
│   ├── auth.test.ts
│   └── images.test.ts
├── .github/
│   └── workflows/
│       └── deploy.yml               # Build + deploy to Lambda on push to main
└── README.md
```

**Infrastructure** lives in `~/code/nexaedge/infrastructure/accounts/management/mdoc-api/`:
```
mdoc-api/
├── main.tf                          # Lambda, API Gateway, S3, IAM roles
├── variables.tf                     # Domain, bucket name, Google OAuth client ID
└── outputs.tf                       # API Gateway URL, S3 bucket name
```

## Configuration

### Config Loading Order

```
1. defaults.yaml (embedded)      ─── base layer, always present
         │
         ▼ deep-merge
2. ~/.config/mdoc/config.yaml    ─── user-level overrides
         │
         ▼ deep-merge
3. ./mdoc.yaml                   ─── project-level overrides
         │
         ▼ deep-merge
4. frontmatter `styles:` block   ─── per-document overrides
         │
         ▼
   Final resolved StyleConfig
```

### Config Implementation with Viper

```go
// internal/config/config.go

func Load() (*StyleConfig, error) {
    v := viper.New()
    v.SetConfigType("yaml")

    // 1. Load embedded defaults
    v.ReadConfig(bytes.NewReader(defaultsYAML))

    // 2. Merge home config
    v.SetConfigName("config")
    v.AddConfigPath("$HOME/.config/mdoc")
    v.MergeInConfig() // ignores "not found"

    // 3. Merge local config
    v.SetConfigName("mdoc")
    v.AddConfigPath(".")
    v.MergeInConfig() // ignores "not found"

    var cfg StyleConfig
    err := v.Unmarshal(&cfg)
    return &cfg, err
}

// 4. Frontmatter merge happens in parser package after extraction
func (cfg *StyleConfig) MergeFromFrontmatter(fm *Frontmatter) {
    // Deep-merge fm.Styles over cfg using reflection or mapstructure
}
```

### StyleConfig Struct

```go
// internal/config/config.go

type StyleConfig struct {
    Drive DriveConfig `yaml:"drive" mapstructure:"drive"`
    Styles Styles     `yaml:"styles" mapstructure:"styles"`
    Code  CodeConfig  `yaml:"code" mapstructure:"code"`
}

type DriveConfig struct {
    DefaultFolder string `yaml:"default_folder" mapstructure:"default_folder"`
}

type Styles struct {
    Page       PageStyle      `yaml:"page" mapstructure:"page"`
    Normal     TextStyle      `yaml:"normal" mapstructure:"normal"`
    Heading1   TextStyle      `yaml:"heading1" mapstructure:"heading1"`
    Heading2   TextStyle      `yaml:"heading2" mapstructure:"heading2"`
    Heading3   TextStyle      `yaml:"heading3" mapstructure:"heading3"`
    Heading4   TextStyle      `yaml:"heading4" mapstructure:"heading4"`
    Heading5   TextStyle      `yaml:"heading5" mapstructure:"heading5"`
    Heading6   TextStyle      `yaml:"heading6" mapstructure:"heading6"`
    CodeInline CodeInlineStyle `yaml:"code_inline" mapstructure:"code_inline"`
    Blockquote BlockquoteStyle `yaml:"blockquote" mapstructure:"blockquote"`
    Link       LinkStyle      `yaml:"link" mapstructure:"link"`
    Table      TableStyle     `yaml:"table" mapstructure:"table"`
    List       ListStyle      `yaml:"list" mapstructure:"list"`
}

type PageStyle struct {
    MarginTop    float64 `yaml:"margin_top" mapstructure:"margin_top"`
    MarginBottom float64 `yaml:"margin_bottom" mapstructure:"margin_bottom"`
    MarginLeft   float64 `yaml:"margin_left" mapstructure:"margin_left"`
    MarginRight  float64 `yaml:"margin_right" mapstructure:"margin_right"`
    PageSize     string  `yaml:"page_size" mapstructure:"page_size"`
}

type TextStyle struct {
    FontFamily    string  `yaml:"font_family" mapstructure:"font_family"`
    FontSize      float64 `yaml:"font_size" mapstructure:"font_size"`
    Bold          bool    `yaml:"bold" mapstructure:"bold"`
    Italic        bool    `yaml:"italic" mapstructure:"italic"`
    Underline     bool    `yaml:"underline" mapstructure:"underline"`
    Strikethrough bool    `yaml:"strikethrough" mapstructure:"strikethrough"`
    Color         string  `yaml:"color" mapstructure:"color"`
    LineSpacing   float64 `yaml:"line_spacing" mapstructure:"line_spacing"`
    SpaceBefore   float64 `yaml:"space_before" mapstructure:"space_before"`
    SpaceAfter    float64 `yaml:"space_after" mapstructure:"space_after"`
    Alignment     string  `yaml:"alignment" mapstructure:"alignment"`
}

type CodeInlineStyle struct {
    FontFamily string `yaml:"font_family" mapstructure:"font_family"`
    Color      string `yaml:"color" mapstructure:"color"` // default color when no language detected
}

type BlockquoteStyle struct {
    TextStyle        `yaml:",inline" mapstructure:",squash"`
    IndentLeft       float64 `yaml:"indent_left" mapstructure:"indent_left"`
    BorderLeftColor  string  `yaml:"border_left_color" mapstructure:"border_left_color"`
    BorderLeftWidth  float64 `yaml:"border_left_width" mapstructure:"border_left_width"`
}

type LinkStyle struct {
    Color     string `yaml:"color" mapstructure:"color"`
    Underline bool   `yaml:"underline" mapstructure:"underline"`
}

type TableStyle struct {
    HeaderBackground    string  `yaml:"header_background" mapstructure:"header_background"`
    HeaderBold          bool    `yaml:"header_bold" mapstructure:"header_bold"`
    BorderColor         string  `yaml:"border_color" mapstructure:"border_color"`
    BorderWidth         float64 `yaml:"border_width" mapstructure:"border_width"`
    CellPadding         float64 `yaml:"cell_padding" mapstructure:"cell_padding"`
    FontSize            float64 `yaml:"font_size" mapstructure:"font_size"`
    VerticalAlignment   string  `yaml:"vertical_alignment" mapstructure:"vertical_alignment"`
    ColumnWidthStrategy string  `yaml:"column_width_strategy" mapstructure:"column_width_strategy"`
}

type ListStyle struct {
    IndentPerLevel    float64 `yaml:"indent_per_level" mapstructure:"indent_per_level"`
    SpaceBetweenItems float64 `yaml:"space_between_items" mapstructure:"space_between_items"`
}

type CodeConfig struct {
    Theme           string  `yaml:"theme" mapstructure:"theme"`
    FontFamily      string  `yaml:"font_family" mapstructure:"font_family"`
    FontSize        float64 `yaml:"font_size" mapstructure:"font_size"`
    CellPadding     float64 `yaml:"cell_padding" mapstructure:"cell_padding"`
    LineNumbers     bool    `yaml:"line_numbers" mapstructure:"line_numbers"`
    BackgroundColor string  `yaml:"background_color" mapstructure:"background_color"`
}
```

### Frontmatter Struct

```go
// internal/parser/frontmatter.go

type Frontmatter struct {
    Title   string                 `yaml:"title"`
    DocID   string                 `yaml:"doc_id"`
    Folder  string                 `yaml:"folder"`
    Account string                 `yaml:"account"`
    TOC     bool                   `yaml:"toc"`
    Styles  map[string]interface{} `yaml:"styles"` // merged into StyleConfig
    Node    *yaml.Node             `yaml:"-"`       // raw yaml.Node tree for lossless writeback
}
```

### Manual Frontmatter Extraction

Frontmatter is extracted by splitting on `---` delimiters and parsing with yaml.v3 Node. This gives us both structured field access and lossless writeback (preserving field order, comments, and unknown fields):

```go
// internal/parser/parser.go

// Parse reads a Markdown file, extracts frontmatter, and parses the body into AST.
func Parse(content []byte) (ast.Node, *Frontmatter, []byte, error) {
    fmBytes, bodyBytes := splitFrontmatter(content)

    var fm Frontmatter
    if fmBytes != nil {
        // Parse into struct for field access
        yaml.Unmarshal(fmBytes, &fm)
        // Also parse as yaml.Node for lossless writeback
        var node yaml.Node
        yaml.Unmarshal(fmBytes, &node)
        fm.Node = &node
    }

    // Parse Markdown body with goldmark
    md := goldmark.New(
        goldmark.WithExtensions(
            extension.Table,
            extension.Strikethrough,
            extension.Footnote,
        ),
    )
    reader := text.NewReader(bodyBytes)
    doc := md.Parser().Parse(reader)

    return doc, &fm, bodyBytes, nil
}

// splitFrontmatter finds --- delimiters and splits content.
// Returns (frontmatterYAML, markdownBody). frontmatterYAML is nil if no frontmatter.
func splitFrontmatter(content []byte) ([]byte, []byte) {
    if !bytes.HasPrefix(content, []byte("---\n")) && !bytes.HasPrefix(content, []byte("---\r\n")) {
        return nil, content
    }
    // Find closing ---
    rest := content[4:] // skip opening "---\n"
    idx := bytes.Index(rest, []byte("\n---\n"))
    if idx == -1 {
        idx = bytes.Index(rest, []byte("\n---\r\n"))
    }
    if idx == -1 {
        return nil, content // no closing delimiter
    }
    fmBytes := rest[:idx]
    bodyBytes := rest[idx+5:] // skip "\n---\n"
    return fmBytes, bodyBytes
}
```

### Frontmatter Preservation Strategy

Writing `doc_id` back to frontmatter without corrupting the file:

```go
// internal/parser/frontmatter.go

func WriteDocID(filePath string, docID string) error {
    content, _ := os.ReadFile(filePath)

    // Find frontmatter boundaries (--- ... ---)
    start, end := findFrontmatterBounds(content)

    if start == -1 {
        // No frontmatter exists — prepend new frontmatter
        newFM := fmt.Sprintf("---\ndoc_id: %q\n---\n", docID)
        return os.WriteFile(filePath, append([]byte(newFM), content...), 0644)
    }

    // Parse existing frontmatter as yaml.Node (preserves structure)
    fmBytes := content[start:end]
    var fmNode yaml.Node
    yaml.Unmarshal(fmBytes, &fmNode)

    // Insert or update doc_id field, preserving order and comments
    setYAMLField(&fmNode, "doc_id", docID)

    // Serialize back and splice into original content
    newFM, _ := yaml.Marshal(&fmNode)
    result := append(content[:start], newFM...)
    result = append(result, content[end:]...)
    return os.WriteFile(filePath, result, 0644)
}
```

**Why yaml.Node:** Using `yaml.Node` from gopkg.in/yaml.v3 preserves field order, comments, and formatting — unlike marshaling through a struct which reorders fields alphabetically and strips comments.

## Converter Architecture — Cursor Pattern

The cursor is the central state machine for the AST→Google Docs conversion. It tracks the current insertion index and generates properly-indexed API requests.

```go
// internal/converter/cursor.go

// Cursor tracks the current insertion position in the Google Doc.
// Google Docs API is index-based: index 1 is the start of body content.
// Every text insertion shifts subsequent indices by the length of inserted text.
type Cursor struct {
    index    int64              // current insertion index (starts at 1)
    requests []*docs.Request   // accumulated API requests
    styles   *ResolvedStyles   // pre-resolved style objects
    strict   bool              // fail on unsupported elements vs. graceful degradation
}

// NewCursor creates a cursor starting at index 1 (beginning of doc body).
func NewCursor(styles *ResolvedStyles, strict bool) *Cursor {
    return &Cursor{index: 1, styles: styles, strict: strict}
}

// InsertText adds text at the current index and advances the cursor.
// Returns the range (startIndex, endIndex) of the inserted text.
func (c *Cursor) InsertText(text string) (start, end int64) {
    start = c.index
    c.requests = append(c.requests, &docs.Request{
        InsertText: &docs.InsertTextRequest{
            Location: &docs.Location{Index: c.index},
            Text:     text,
        },
    })
    c.index += int64(len([]rune(text))) // Google Docs counts Unicode code points
    end = c.index
    return start, end
}

// ApplyTextStyle applies a text style to a range.
func (c *Cursor) ApplyTextStyle(start, end int64, style *docs.TextStyle, fields string) {
    c.requests = append(c.requests, &docs.Request{
        UpdateTextStyle: &docs.UpdateTextStyleRequest{
            Range:     &docs.Range{StartIndex: start, EndIndex: end},
            TextStyle: style,
            Fields:    fields, // e.g., "bold,foregroundColor,weightedFontFamily"
        },
    })
}

// ApplyParagraphStyle applies a paragraph style to a range.
func (c *Cursor) ApplyParagraphStyle(start, end int64, style *docs.ParagraphStyle, fields string) {
    c.requests = append(c.requests, &docs.Request{
        UpdateParagraphStyle: &docs.UpdateParagraphStyleRequest{
            Range:          &docs.Range{StartIndex: start, EndIndex: end},
            ParagraphStyle: style,
            Fields:         fields,
        },
    })
}

// InsertTable inserts a table and returns the table start index.
// After insertion, the cursor is positioned inside the first cell.
func (c *Cursor) InsertTable(rows, cols int) int64 {
    tableStart := c.index
    c.requests = append(c.requests, &docs.Request{
        InsertTable: &docs.InsertTableRequest{
            Location: &docs.Location{Index: c.index},
            Rows:     int64(rows),
            Columns:  int64(cols),
        },
    })
    // Table structure adds indices: table start, row start, cell start, paragraph start
    // For a 1x1 table: table(+1), row(+1), cell(+1), paragraph(+0) = cursor at index+3
    // Actual offset depends on table size — must be calculated per Google Docs API behavior
    c.index += 4 // for 1x1: table + row + cell + newline in cell
    return tableStart
}

// Requests returns the accumulated API requests.
func (c *Cursor) Requests() []*docs.Request {
    return c.requests
}
```

### Node Handler Pattern

Each Markdown element type has a dedicated handler function:

```go
// internal/converter/converter.go

// Convert walks the AST and generates Google Docs API requests.
func Convert(node ast.Node, source []byte, styles *ResolvedStyles, strict bool) ([]*docs.Request, error) {
    c := NewCursor(styles, strict)
    err := ast.Walk(node, func(n ast.Node, entering bool) (ast.WalkStatus, error) {
        switch n.Kind() {
        case ast.KindParagraph:
            return handleParagraph(c, n, source, entering)
        case ast.KindHeading:
            return handleHeading(c, n.(*ast.Heading), source, entering)
        case ast.KindCodeBlock, ast.KindFencedCodeBlock:
            return handleCodeBlock(c, n, source, entering)
        case ast.KindList:
            return handleList(c, n.(*ast.List), source, entering)
        case ast.KindBlockquote:
            return handleBlockquote(c, n, source, entering)
        case extast.KindTable:
            return handleTable(c, n, source, entering)
        case ast.KindThematicBreak:
            return ast.WalkSkipChildren, nil // discard horizontal rules
        // ... inline nodes handled during text extraction
        default:
            if strict {
                return ast.WalkStop, fmt.Errorf("unsupported node: %s", n.Kind())
            }
            return ast.WalkContinue, nil
        }
    })
    return c.Requests(), err
}
```

### Code Block Handler (Key Complexity)

```go
// internal/converter/code.go

func handleCodeBlock(c *Cursor, node ast.Node, source []byte, entering bool) (ast.WalkStatus, error) {
    if !entering {
        return ast.WalkContinue, nil
    }

    fenced, ok := node.(*ast.FencedCodeBlock)
    lang := ""
    if ok && fenced.Language(source) != nil {
        lang = string(fenced.Language(source))
    }

    // Extract code text from node lines
    code := extractCodeText(node, source)

    // 1. Insert 1x1 table
    tableStart := c.InsertTable(1, 1)

    // 2. Tokenize with chroma
    lexer := lexers.Get(lang)
    if lexer == nil {
        lexer = lexers.Fallback
    }
    lexer = chroma.Coalesce(lexer) // merge adjacent same-type tokens
    iterator, _ := lexer.Tokenise(nil, code)

    // 3. Insert each token with styling
    tokens := iterator.Tokens()
    for _, token := range mergeAdjacentSameStyle(tokens, c.styles) {
        start, end := c.InsertText(token.Value)
        tokenStyle := c.styles.TokenStyle(token.Type)
        if tokenStyle != nil {
            c.ApplyTextStyle(start, end, tokenStyle, tokenStyleFields(token.Type))
        }
    }

    // 4. Apply cell background
    c.ApplyTableCellStyle(tableStart, &docs.TableCellStyle{
        BackgroundColor: c.styles.CodeBackground(),
        ContentAlignment: "TOP",
    })

    // 5. Advance cursor past table end
    c.AdvancePastTable()

    return ast.WalkSkipChildren, nil
}

// mergeAdjacentSameStyle reduces API request count by combining
// adjacent tokens that would receive identical styling.
func mergeAdjacentSameStyle(tokens []chroma.Token, styles *ResolvedStyles) []chroma.Token {
    if len(tokens) == 0 {
        return tokens
    }
    merged := []chroma.Token{tokens[0]}
    for _, t := range tokens[1:] {
        last := &merged[len(merged)-1]
        if styles.SameTokenStyle(last.Type, t.Type) {
            last.Value += t.Value
        } else {
            merged = append(merged, t)
        }
    }
    return merged
}
```

## Google Docs API Client

### Interface for Testability

```go
// internal/gdocs/client.go

// DocsClient abstracts the Google Docs API for testability.
type DocsClient interface {
    // CreateDocument creates a new empty Google Doc.
    CreateDocument(ctx context.Context, title string) (*docs.Document, error)

    // GetDocument retrieves a document (used for upsert confirmation title).
    GetDocument(ctx context.Context, docID string) (*docs.Document, error)

    // BatchUpdate sends a list of requests to modify a document.
    // Handles splitting into chunks if the request list exceeds API limits.
    BatchUpdate(ctx context.Context, docID string, requests []*docs.Request) error

    // DeleteContent clears all body content from a document (for upsert).
    DeleteContent(ctx context.Context, docID string) error
}

// DriveClient abstracts the Google Drive API.
type DriveClient interface {
    // MoveToFolder moves a file to the specified folder, creating folders as needed.
    MoveToFolder(ctx context.Context, fileID string, folderPath string) error

    // EnsureFolder creates a folder path if it doesn't exist, returns the leaf folder ID.
    EnsureFolder(ctx context.Context, path string) (string, error)
}

// realDocsClient implements DocsClient using the official Google API.
type realDocsClient struct {
    svc       *docs.Service
    limiter   *RateLimiter
}

// mockDocsClient implements DocsClient for testing.
type mockDocsClient struct {
    requests [][]*docs.Request // captured batchUpdate calls
    docs     map[string]*docs.Document
}
```

### Rate Limiting

```go
// internal/gdocs/ratelimit.go

// RateLimiter enforces proactive RPS limits before hitting Google API quotas.
// Uses a token bucket algorithm.
type RateLimiter struct {
    limiter *rate.Limiter // golang.org/x/time/rate
    backoff *ExponentialBackoff
}

// Google Docs API default quota: 300 requests per minute per user = 5 RPS
// We use 4 RPS to leave headroom.
func NewRateLimiter() *RateLimiter {
    return &RateLimiter{
        limiter: rate.NewLimiter(rate.Limit(4), 1), // 4 req/sec, burst of 1
        backoff: &ExponentialBackoff{
            InitialInterval: 1 * time.Second,
            MaxInterval:     60 * time.Second,
            Multiplier:      2.0,
            Jitter:          0.1,
        },
    }
}

// Wait blocks until the rate limiter allows the next request.
func (r *RateLimiter) Wait(ctx context.Context) error {
    return r.limiter.Wait(ctx)
}

// DoWithRetry executes fn with rate limiting and exponential backoff on 429/5xx.
func (r *RateLimiter) DoWithRetry(ctx context.Context, fn func() error) error {
    for attempt := 0; attempt < 5; attempt++ {
        r.Wait(ctx)
        err := fn()
        if err == nil {
            return nil
        }
        if isRetryable(err) {
            time.Sleep(r.backoff.Next(attempt))
            continue
        }
        return err
    }
    return fmt.Errorf("max retries exceeded")
}
```

### Request Batching

```go
// internal/gdocs/batch.go

// MaxBatchSize is the maximum number of requests per batchUpdate call.
// Google Docs API doesn't document a hard limit, but large batches can
// timeout. 500 requests per batch is a safe practical limit.
const MaxBatchSize = 500

// SplitBatch divides a request list into chunks.
// Index correctness is maintained because requests are ordered sequentially
// and each batch picks up where the previous left off.
func SplitBatch(requests []*docs.Request) [][]*docs.Request {
    var batches [][]*docs.Request
    for i := 0; i < len(requests); i += MaxBatchSize {
        end := i + MaxBatchSize
        if end > len(requests) {
            end = len(requests)
        }
        batches = append(batches, requests[i:end])
    }
    return batches
}
```

## Authentication Flow

```
User                    mdoc CLI                     mdoc-api                Google
  │                        │                            │                      │
  │  mdoc auth             │                            │                      │
  │───────────────────────►│                            │                      │
  │                        │  1. start local server     │                      │
  │                        │     on random port         │                      │
  │                        │                            │                      │
  │                        │  2. open browser to:       │                      │
  │  ◄── opens browser ────│  /auth/start?redirect_uri= │                      │
  │  (browser navigates)   │  http://localhost:{port}   │                      │
  │                        │  /callback                 │                      │
  │                        │                            │                      │
  │                        │              GET /auth/start│                      │
  │                        │  ──────────────────────────►│  3. redirect to     │
  │                        │                            │     Google OAuth     │
  │                        │                            │─────────────────────►│
  │                        │                            │                      │
  │  4. user logs in + consents on Google               │  ◄── auth code      │
  │                        │                            │◄─────────────────────│
  │                        │                            │                      │
  │                        │                            │  5. exchange code    │
  │                        │                            │─────────────────────►│
  │                        │                            │  ◄── tokens          │
  │                        │                            │◄─────────────────────│
  │                        │                            │                      │
  │                        │  6. redirect browser to    │                      │
  │                        │     localhost:{port}/callback                     │
  │                        │     with tokens in query   │                      │
  │                        │◄───────────────────────────│                      │
  │                        │                            │                      │
  │                        │  7. store token at         │                      │
  │                        │  ~/.config/mdoc/accounts/  │                      │
  │                        │  <name>/token.json         │                      │
  │                        │  8. shut down local server │                      │
  │  ◄── "Authenticated!" │                            │                      │
  │◄───────────────────────│                            │                      │
```

### Token Storage

```go
// internal/auth/token.go

const configDir = "~/.config/mdoc"

// TokenPath returns the path for a specific account's token file.
func TokenPath(account string) string {
    if account == "" {
        account = "default"
    }
    return filepath.Join(expandHome(configDir), "accounts", account, "token.json")
}

// LoadToken reads and returns the OAuth2 token for an account.
func LoadToken(account string) (*oauth2.Token, error) {
    data, err := os.ReadFile(TokenPath(account))
    if err != nil {
        return nil, fmt.Errorf("not authenticated (run 'mdoc auth'): %w", err)
    }
    var token oauth2.Token
    json.Unmarshal(data, &token)
    return &token, nil
}

// SaveToken writes an OAuth2 token for an account.
func SaveToken(account string, token *oauth2.Token) error {
    path := TokenPath(account)
    os.MkdirAll(filepath.Dir(path), 0700)
    data, _ := json.MarshalIndent(token, "", "  ")
    return os.WriteFile(path, data, 0600) // restrictive permissions
}
```

## Style Resolution

```go
// internal/styles/resolver.go

// ResolvedStyles contains pre-computed Google Docs API style objects.
type ResolvedStyles struct {
    Page        *docs.DocumentStyle
    Normal      resolvedTextAndParagraph
    Headings    [6]resolvedTextAndParagraph // index 0 = H1
    CodeInline  *docs.TextStyle
    Blockquote  resolvedBlockquote
    Link        *docs.TextStyle
    Table       resolvedTable
    List        resolvedList
    CodeBlock   resolvedCodeBlock
    TokenStyles map[chroma.TokenType]*docs.TextStyle // pre-computed from theme
}

type resolvedTextAndParagraph struct {
    Text      *docs.TextStyle
    Paragraph *docs.ParagraphStyle
    Fields    string // UpdateTextStyle fields mask
    PFields   string // UpdateParagraphStyle fields mask
}

// Resolve converts a StyleConfig into Google Docs API objects.
func Resolve(cfg *config.StyleConfig) (*ResolvedStyles, error) {
    rs := &ResolvedStyles{}

    // Convert hex colors to Google Docs RGB floats
    // e.g., "#1a1a1a" → &docs.Color{RgbColor: &docs.RgbColor{Red: 0.102, Green: 0.102, Blue: 0.102}}
    rs.Normal = resolveTextStyle(cfg.Styles.Normal)
    for i, h := range []config.TextStyle{
        cfg.Styles.Heading1, cfg.Styles.Heading2, cfg.Styles.Heading3,
        cfg.Styles.Heading4, cfg.Styles.Heading5, cfg.Styles.Heading6,
    } {
        rs.Headings[i] = resolveTextStyle(h)
    }

    // Resolve code theme
    theme := styles.Get(cfg.Code.Theme)
    if theme == nil {
        return nil, fmt.Errorf("unknown chroma theme: %s", cfg.Code.Theme)
    }
    rs.TokenStyles = resolveTokenStyles(theme)
    rs.CodeBlock = resolveCodeBlock(cfg.Code, theme)

    return rs, nil
}

// hexToColor converts "#rrggbb" to Google Docs OptionalColor.
func hexToColor(hex string) *docs.OptionalColor {
    r, g, b := parseHex(hex)
    return &docs.OptionalColor{
        Color: &docs.Color{
            RgbColor: &docs.RgbColor{
                Red:   float64(r) / 255.0,
                Green: float64(g) / 255.0,
                Blue:  float64(b) / 255.0,
            },
        },
    }
}
```

## Table Column Width Algorithm

```go
// internal/converter/table.go

type columnClass int
const (
    fitToContent  columnClass = iota
    proportional
)

// classifyColumns analyzes table content to determine column width strategy.
func classifyColumns(table *extast.Table, source []byte) []columnClass {
    // For each column, scan all cell values
    classes := make([]columnClass, table.Columns)
    for col := 0; col < table.Columns; col++ {
        maxLen := 0
        allShort := true
        for _, row := range table.Rows {
            cellText := extractCellText(row.Cells[col], source)
            if len(cellText) > 20 { // threshold: 20 chars
                allShort = false
            }
            if len(cellText) > maxLen {
                maxLen = len(cellText)
            }
        }
        if allShort || looksNumeric(col, table, source) {
            classes[col] = fitToContent
        } else {
            classes[col] = proportional
        }
    }
    return classes
}

// calculateWidths returns column widths in points, given total table width.
func calculateWidths(classes []columnClass, maxContentWidths []int, totalWidth float64) []float64 {
    // 1. Estimate fit-to-content widths (char count * avg char width + padding)
    // 2. Sum fit-to-content widths
    // 3. Distribute remaining width proportionally among proportional columns
    // 4. If all columns are proportional, divide equally
    // ...
}
```

## Testing Strategy

### Golden File Tests

Each test case is a directory under `testdata/` containing:
- `input.md` — Markdown input (may include frontmatter)
- `config.yaml` — Optional per-test config overrides
- `expected.json` — Expected list of `docs.Request` objects (serialized as JSON)

```go
// internal/converter/converter_test.go

func TestConverter(t *testing.T) {
    entries, _ := os.ReadDir("../../testdata")
    for _, entry := range entries {
        if !entry.IsDir() {
            continue
        }
        t.Run(entry.Name(), func(t *testing.T) {
            dir := filepath.Join("../../testdata", entry.Name())

            // Load input
            input, _ := os.ReadFile(filepath.Join(dir, "input.md"))

            // Load optional config
            cfg := config.LoadDefaults()
            if cfgFile, err := os.ReadFile(filepath.Join(dir, "config.yaml")); err == nil {
                cfg.MergeFrom(cfgFile)
            }

            // Parse and convert
            ast, fm, _ := parser.Parse(input)
            styles, _ := styles.Resolve(cfg)
            requests, _ := converter.Convert(ast, input, styles, true)

            // Compare with golden file
            expected, _ := os.ReadFile(filepath.Join(dir, "expected.json"))
            actual, _ := json.MarshalIndent(requests, "", "  ")

            if !bytes.Equal(actual, expected) {
                // Write actual output for diff
                os.WriteFile(filepath.Join(dir, "actual.json"), actual, 0644)
                t.Errorf("output mismatch (see actual.json)\n%s", diff(expected, actual))
            }
        })
    }
}
```

### Update Golden Files

```bash
# Regenerate all golden files (when intentionally changing output)
GOLDEN_UPDATE=1 go test ./internal/converter/...
```

### Test Categories

| Category | What | How |
|----------|------|-----|
| **Unit: Cursor** | Index math is correct | Direct cursor manipulation, assert index values |
| **Unit: Node handlers** | Each Markdown element produces correct requests | Golden files per element type |
| **Unit: Config merge** | Layered config merging works | Multiple config files, assert merged values |
| **Unit: Frontmatter** | Preservation on writeback | Write doc_id, assert rest unchanged |
| **Unit: Style resolver** | Hex→RGB, theme→token styles | Known inputs, expected API style objects |
| **Unit: Column widths** | Smart width classification | Tables with known content, assert widths |
| **Integration: Full pipeline** | End-to-end from Markdown to request list | Complex golden files with all element types |
| **Integration: API mock** | Batching, rate limiting, upsert flow | Mock DocsClient, verify call sequences |

## Image Upload Flow

```go
// internal/images/upload.go

// Upload sends an image to mdoc-api and returns a public URL.
// Works for both local files and remote URLs.
func Upload(ctx context.Context, apiBaseURL string, token string, src string) (string, error) {
    var body io.Reader
    var filename string

    if isURL(src) {
        // Download image first, then upload to mdoc-api
        resp, _ := http.Get(src)
        defer resp.Body.Close()
        body = resp.Body
        filename = filepath.Base(src)
    } else {
        // Local file
        f, _ := os.Open(src)
        defer f.Close()
        body = f
        filename = filepath.Base(src)
    }

    // Multipart upload to mdoc-api
    var buf bytes.Buffer
    writer := multipart.NewWriter(&buf)
    part, _ := writer.CreateFormFile("image", filename)
    io.Copy(part, body)
    writer.Close()

    req, _ := http.NewRequestWithContext(ctx, "POST", apiBaseURL+"/images/upload", &buf)
    req.Header.Set("Content-Type", writer.FormDataContentType())
    req.Header.Set("Authorization", "Bearer "+token)

    resp, _ := http.DefaultClient.Do(req)
    var result struct {
        URL string `json:"url"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.URL, nil
}
```

## mdoc-api Architecture

### Hono Application

```typescript
// src/index.ts
import { Hono } from "hono";
import { handle } from "@hono/aws-lambda";
import { authRoutes } from "./routes/auth";
import { imageRoutes } from "./routes/images";
import { healthRoutes } from "./routes/health";

const app = new Hono();

app.route("/auth", authRoutes);
app.route("/images", imageRoutes);
app.route("/", healthRoutes);

export const handler = handle(app);
```

### OAuth2 Routes

```typescript
// src/routes/auth.ts
import { Hono } from "hono";
import { OAuth2Client } from "google-auth-library";
import { getConfig } from "../config";

export const authRoutes = new Hono();

// Step 1: CLI calls /auth/start → redirect to Google consent screen
authRoutes.get("/start", async (c) => {
  const redirectUri = c.req.query("redirect_uri"); // e.g., http://localhost:9876/callback
  if (!redirectUri) {
    return c.json({ error: "redirect_uri required" }, 400);
  }

  const config = getConfig();
  const oauth2Client = new OAuth2Client(
    config.googleClientId,
    config.googleClientSecret,
    `${config.apiBaseUrl}/auth/callback`
  );

  // Store redirect_uri in state param so we can retrieve it in callback
  const state = Buffer.from(JSON.stringify({ redirectUri })).toString("base64url");

  const authUrl = oauth2Client.generateAuthUrl({
    access_type: "offline",
    prompt: "consent",
    scope: [
      "https://www.googleapis.com/auth/documents",
      "https://www.googleapis.com/auth/drive.file",
    ],
    state,
  });

  return c.redirect(authUrl);
});

// Step 2: Google redirects here with auth code → exchange → redirect to CLI
authRoutes.get("/callback", async (c) => {
  const code = c.req.query("code");
  const stateParam = c.req.query("state");

  if (!code || !stateParam) {
    return c.json({ error: "missing code or state" }, 400);
  }

  const { redirectUri } = JSON.parse(
    Buffer.from(stateParam, "base64url").toString()
  );

  const config = getConfig();
  const oauth2Client = new OAuth2Client(
    config.googleClientId,
    config.googleClientSecret,
    `${config.apiBaseUrl}/auth/callback`
  );

  // Exchange code for tokens
  const { tokens } = await oauth2Client.getToken(code);

  // Redirect to CLI's local callback with tokens
  const params = new URLSearchParams({
    access_token: tokens.access_token!,
    refresh_token: tokens.refresh_token || "",
    expiry_date: String(tokens.expiry_date || ""),
    token_type: tokens.token_type || "Bearer",
  });

  return c.redirect(`${redirectUri}?${params.toString()}`);
});
```

### Image Upload Routes

```typescript
// src/routes/images.ts
import { Hono } from "hono";
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { randomUUID } from "crypto";
import { getConfig } from "../config";

export const imageRoutes = new Hono();

const s3 = new S3Client({});

// Upload an image → store in S3 → return public URL
imageRoutes.post("/upload", async (c) => {
  const authHeader = c.req.header("Authorization");
  if (!authHeader?.startsWith("Bearer ")) {
    return c.json({ error: "unauthorized" }, 401);
  }

  const body = await c.req.parseBody();
  const file = body["file"];
  if (!(file instanceof File)) {
    return c.json({ error: "file required" }, 400);
  }

  const config = getConfig();
  const ext = file.name.split(".").pop() || "png";
  const key = `images/${randomUUID()}.${ext}`;

  await s3.send(
    new PutObjectCommand({
      Bucket: config.s3Bucket,
      Key: key,
      Body: Buffer.from(await file.arrayBuffer()),
      ContentType: file.type || "image/png",
    })
  );

  const url = `${config.apiBaseUrl}/images/${key.replace("images/", "")}`;
  return c.json({ url });
});

// Serve an image from S3 (public, used by Google Docs to fetch)
imageRoutes.get("/:id", async (c) => {
  const id = c.req.param("id");
  const config = getConfig();

  try {
    const response = await s3.send(
      new GetObjectCommand({
        Bucket: config.s3Bucket,
        Key: `images/${id}`,
      })
    );

    const body = await response.Body?.transformToByteArray();
    if (!body) return c.notFound();

    return new Response(body, {
      headers: {
        "Content-Type": response.ContentType || "image/png",
        "Cache-Control": "public, max-age=600",
      },
    });
  } catch {
    return c.notFound();
  }
});
```

### Configuration

```typescript
// src/config.ts

export interface Config {
  googleClientId: string;
  googleClientSecret: string;
  s3Bucket: string;
  apiBaseUrl: string;
}

export function getConfig(): Config {
  return {
    googleClientId: process.env.GOOGLE_CLIENT_ID!,
    googleClientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    s3Bucket: process.env.S3_BUCKET!,
    apiBaseUrl: process.env.API_BASE_URL!, // e.g., https://api.mdoc.dev
  };
}
```

### Infrastructure (Terraform)

Key resources in `~/code/nexaedge/infrastructure/accounts/management/mdoc-api/main.tf`:

```hcl
# S3 bucket for ephemeral image hosting
resource "aws_s3_bucket" "images" {
  bucket = "mdoc-api-images"
}

resource "aws_s3_bucket_lifecycle_configuration" "images_ttl" {
  bucket = aws_s3_bucket.images.id

  rule {
    id     = "expire-images"
    status = "Enabled"

    expiration {
      days = 1  # S3 minimum granularity. Images live up to 24h.
    }
  }
}

# Lambda function
resource "aws_lambda_function" "mdoc_api" {
  function_name = "mdoc-api"
  runtime       = "nodejs22.x"
  handler       = "index.handler"
  timeout       = 30  # seconds (image upload may take time)
  memory_size   = 256 # MB

  environment {
    variables = {
      GOOGLE_CLIENT_ID     = var.google_client_id
      GOOGLE_CLIENT_SECRET = var.google_client_secret
      S3_BUCKET            = aws_s3_bucket.images.id
      API_BASE_URL         = "https://${var.domain}"
    }
  }
}

# API Gateway v2 (HTTP API)
resource "aws_apigatewayv2_api" "mdoc_api" {
  name          = "mdoc-api"
  protocol_type = "HTTP"
}

# Route53 + ACM for custom domain (e.g., api.mdoc.dev)
# ... (uses existing NexaEdge patterns)
```

### CLI ↔ mdoc-api: Auth Flow (Updated)

The auth flow uses a local HTTP server on the CLI side:

```go
// internal/auth/auth.go (CLI side)

func Authenticate(account string, apiBaseURL string) error {
    // 1. Start local callback server on a random available port
    listener, _ := net.Listen("tcp", "localhost:0")
    port := listener.Addr().(*net.TCPAddr).Port
    callbackURL := fmt.Sprintf("http://localhost:%d/callback", port)

    tokenCh := make(chan *oauth2.Token, 1)
    errCh := make(chan error, 1)

    // 2. Handle the callback from mdoc-api
    mux := http.NewServeMux()
    mux.HandleFunc("/callback", func(w http.ResponseWriter, r *http.Request) {
        accessToken := r.URL.Query().Get("access_token")
        refreshToken := r.URL.Query().Get("refresh_token")
        expiryDate := r.URL.Query().Get("expiry_date")

        if accessToken == "" {
            errCh <- fmt.Errorf("no access_token in callback")
            fmt.Fprint(w, "Authorization failed. You can close this tab.")
            return
        }

        expiry, _ := strconv.ParseInt(expiryDate, 10, 64)
        token := &oauth2.Token{
            AccessToken:  accessToken,
            RefreshToken: refreshToken,
            Expiry:       time.UnixMilli(expiry),
            TokenType:    "Bearer",
        }
        tokenCh <- token
        fmt.Fprint(w, "Authorization successful! You can close this tab.")
    })

    srv := &http.Server{Handler: mux}
    go srv.Serve(listener)
    defer srv.Close()

    // 3. Open browser to mdoc-api /auth/start
    authURL := fmt.Sprintf("%s/auth/start?redirect_uri=%s",
        apiBaseURL, url.QueryEscape(callbackURL))
    openBrowser(authURL)

    fmt.Printf("Waiting for authorization (listening on port %d)...\n", port)

    // 4. Wait for token or error
    select {
    case token := <-tokenCh:
        SaveToken(account, token)
        fmt.Println("Authorization successful!")
        return nil
    case err := <-errCh:
        return err
    case <-time.After(5 * time.Minute):
        return fmt.Errorf("authorization timed out")
    }
}
```

## Dependencies

```
# go.mod — github.com/jaisonerick/mdoc

go 1.26

require (
    # CLI
    github.com/spf13/cobra      v1.9.1     # Context7 documented; latest is v1.10.2
    github.com/spf13/viper      v1.20.1    # Context7 documented; latest is v1.21.0

    # Markdown parsing
    github.com/yuin/goldmark    v1.7.16    # latest; Context7 has docs (no version-specific)

    # Syntax highlighting
    github.com/alecthomas/chroma/v2  v2.23.1  # latest; Context7 has docs (no version-specific)

    # Google APIs (docs/v1, drive/v3 — same module)
    google.golang.org/api       v0.270.0   # latest; requires Go 1.25+

    # OAuth2
    golang.org/x/oauth2         v0.36.0    # latest; google-api requires >= v0.35.0

    # Rate limiting
    golang.org/x/time           v0.15.0    # latest; no Context7 docs

    # YAML (frontmatter preservation via yaml.Node)
    gopkg.in/yaml.v3            v3.0.1     # latest; Context7 has docs
)

# Frontmatter parsed manually (--- splitting + yaml.v3 Node) for lossless writeback.
# See internal/parser package.

# Transitive dependencies of note (from google.golang.org/api):
#   golang.org/x/net   v0.51.0+
#   golang.org/x/sync  v0.19.0+
#   cloud.google.com/go/auth v0.18.2+
#
# Transitive from viper:
#   github.com/spf13/pflag      v1.0.10  (also used by cobra — compatible)
#   go.yaml.in/yaml/v3          v3.0.4   (new import path, coexists with gopkg.in/yaml.v3)
#   github.com/go-viper/mapstructure/v2  v2.4.0
```

### mdoc-api Dependencies

```json
// package.json — github.com/jaisonerick/mdoc-api
{
  "dependencies": {
    "hono": "^4.x",
    "@hono/aws-lambda": "^1.x",
    "@aws-sdk/client-s3": "^3.x",
    "google-auth-library": "^9.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "vitest": "^3.x",
    "esbuild": "^0.25.x",
    "@types/aws-lambda": "^8.x"
  }
}
```

## Key Design Decisions Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Language | Go 1.26 | Required by google.golang.org/api v0.270.0. Single-binary distribution, matches plaud-cli pattern. |
| CLI framework | Cobra | Industry standard, subcommand routing, pairs with Viper. Same as plaud-cli. |
| Config library | Viper | Built-in layered merge, YAML support, Cobra integration |
| Markdown parser | goldmark | CommonMark-compliant, extensible, built-in table/footnote extensions, pure Go |
| Frontmatter | Manual split + yaml.v3 Node | Manual `---` splitting + yaml.v3 Node gives lossless writeback preserving field order, comments, and unknown fields. |
| Syntax highlighting | chroma v2 | Same as Hugo, 200+ languages, 40+ themes, token iterator API for per-token control |
| Google API client | Official Go client | Type-safe, handles auth refresh, well-maintained |
| Index tracking | Cursor pattern | Mutable cursor passed through AST walker. Explicit, testable, each handler advances cursor. |
| Testing | Golden files + mock API | Deterministic, catches regressions, fast. Mock interface for DocsClient. |
| Frontmatter writeback | yaml.Node manipulation | Preserves field order, comments, formatting. Doesn't corrupt existing content. |
| Config format | YAML | Matches frontmatter format, familiar, good for nested style definitions |
| Distribution | GoReleaser + GitHub Releases | Same as plaud-cli. Cross-compilation, checksums, curl installer. |
| Rate limiting | Token bucket (4 RPS) + exponential backoff | Proactive limit below quota (5 RPS) as first defense, backoff as fallback |
| Request batching | 500 requests per batchUpdate | Safe practical limit. Sequential ordering preserves index correctness. |
| mdoc-api language | TypeScript + Hono | Best DX for a small Lambda service. Fast cold starts on Node.js 22. Hono runs natively on Lambda with minimal boilerplate. |
| mdoc-api deployment | Lambda + API Gateway v2 | Serverless, pay-per-use. Zero cost when idle. 3 endpoints don't justify an always-on service. |
| Image storage | S3 with 1-day lifecycle | S3 lifecycle minimum is 1 day. Acceptable for ephemeral images — low volume, no privacy concern. Simpler than Lambda cron cleanup. |
| Image serving | API proxy to S3 | Serve via `/images/:id` endpoint rather than direct S3 URLs. Keeps S3 bucket private, allows future access control or CDN. |
| OAuth callback | Local HTTP server (dynamic port) | CLI starts a temporary server on a random port. More reliable than fixed port (no conflicts). 5-minute timeout. |
| mdoc-api infrastructure | Terraform in NexaEdge infra repo | Consistent with existing infrastructure management. Lambda, API Gateway, S3, Route53 all managed centrally. |
| mdoc-api auth | No API key | OAuth start is public (Google login required). Image upload requires Google access token. Low-traffic service doesn't need additional gating. |

## Open Questions Resolved

| Question (from spec) | Resolution |
|----------------------|------------|
| Inline code language detection | Use chroma's `lexers.Analyse()` for heuristic detection. If confidence is low, fall back to configurable default color. Future: frontmatter field for default inline language. |
| Table width hints in Markdown | Start with smart auto-detection only. If insufficient, add `<!-- mdoc:widths -->` comment syntax in a future version. |
| mdoc-api hosting | TypeScript + Hono on Lambda + API Gateway v2. Infrastructure in NexaEdge Terraform repo. S3 for image storage with 1-day lifecycle. Separate repo (`jaisonerick/mdoc-api`). |
| Image size/scaling | Default: scale to page width if wider than margins. Future: per-image size hints via alt-text syntax (e.g., `![alt\|50%](url)`). |
| Nested blockquotes | Increase indent by `indent_left` per nesting level. Single border color for all levels. |
| TOC placement | Insert after document title (first element). Future: support `<!-- toc -->` marker. |
