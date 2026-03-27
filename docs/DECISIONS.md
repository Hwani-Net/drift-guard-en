# drift-guard — Architecture Decision Records

## ADR-001: Project Idea Selection — "Design Guard" (2026-03-11)

### Context
We needed to develop a project to qualify simultaneously for the OpenAI + Anthropic open source programs.
Among 6 candidate ideas (awesome-mcp-skills, mcpx, ai-rules, stitch-to-code, agent-guard, nongjong-agent rebrand), interest was shown in Idea D "stitch-to-code", but the question arose: "Is this only my problem?"

### Research Findings
- Design Drift (AI destroying designs when adding features) is the biggest industry issue in 2026
- Design Systems Collective: "Design drift is becoming the biggest cost"
- Reddit r/FigmaDesign: "Design-to-Code handoff still incredibly broken in 2026"
- Stitch official forum: 30+ version frustration cases documented
- **No direct competitors** (BackstopJS/Percy are post-hoc QA tools, different category)

### Decision
Pivot from "design-to-code converter" to "AI coding drift detection and blocking CLI".
Project name: `drift-guard`

### Rationale
1. v0/Bolt/Figma Make have billions invested in converters — cannot compete
2. "Detect and block" is achievable with a single CLI — launchable solo within 3.5 months
3. No direct competitor — first-mover advantage
4. AI agent rule file generation is the key differentiator — applicable across the entire AI coding tool ecosystem

---

## ADR-002: Technology Stack Selection (2026-03-11)

### Decision
- **Language**: TypeScript (strict mode)
- **CSS Parser**: css-tree (instead of PostCSS)
- **HTML Parser**: cheerio
- **Build**: tsup
- **Testing**: Vitest
- **Distribution**: npm + GitHub

### Rationale
- css-tree: Lighter than PostCSS, AST walking is more intuitive
- cheerio: 10x faster than jsdom, optimal for server-side HTML parsing
- tsup: esbuild-based, 17ms build time. No rollup/webpack needed
- npm distribution: `npx drift-guard init` works instantly — favorable for virality

---

## ADR-003: GitHub Account Strategy (2026-03-11)

### Decision
Use existing GitHub account (no new account creation)

### Rationale
- OpenAI/Anthropic programs: require "evidence of active maintenance"
- New account = zero commit history = disadvantageous in review
- GitHub TOS: recommends one person = one personal account
- Cleaning up existing account profile + repos is more effective

---

## ADR-004: Adding Tailwind Config Parsing for Stitch HTML (2026-03-12)

### Context
Stitch outputs design tokens (colors, radius, fonts) not in CSS style tags but inside a script tag with id tailwind-config as a JS object. The drift-guard HTML parser only parsed style tags, so it could not detect changes to Stitch design primary colors (#256af4) — a critical bug.

### Decision
Add `extractTailwindConfig()` function to `html-parser.ts`. Find `tailwind.config` or `id="tailwind-config"` in script tags and use regex to parse colors, borderRadius, fontFamily, converting them to `--tw-*` prefixed design tokens.

### Rationale
- Stitch using Tailwind CDN + inline config is Stitch standard output format
- Separate extraction function is better for separation of concerns than modifying the CSS parser
- `--tw-` prefix prevents naming conflicts with CSS variable tokens
- Regex-based implementation keeps it lightweight without JS parser dependencies

---

## ADR-005: DOM Structure Change Detection — sha256 Fingerprint (2026-03-12)

### Context
drift-guard v0.1.x only tracked CSS tokens (colors, fonts, spacing, etc.). If AI changed HTML structure (div nesting changes, semantic tag deletion), it could not detect anything. Tracking DOM structure is essential to claim "complete design protection."

### Decision
New `structure-parser.ts` module calculates 4 fingerprints using cheerio DFS:
1. `semanticTags`: count of header/nav/main/section/article/aside/footer/form/table/dialog
2. `maxDepth`: maximum DOM nesting depth
3. `layoutHash`: tagName.className of flex/grid display elements — sha256 first 8 chars
4. `childSequenceHash`: body direct children tag sequence — sha256 first 8 chars

### Rationale
- Full DOM tree serialization comparison is too costly with whitespace noise issues
- Fingerprint method determines "did the structure change?" in O(1)
- Full tree diff (not sha256 hash) to be considered in v0.3.0
- `structure` field kept optional for backward compatibility with v0.1.x snapshots

---

## ADR-006: Separating MCP Wrapper as a Separate Package (2026-03-12)

### Context
Need to expose drift-guard Programmatic API as an MCP server for direct use in Cursor/Claude Code/Codex. Package structure options: (A) add entry point to existing package, (B) separate npm package.

### Decision
Separate `@stayicon/drift-guard-mcp` package (Option B). `drift-guard-mcp/` subdirectory.

### Rationale
1. CLI users do not need `@modelcontextprotocol/sdk` + `zod` dependencies — saves install size
2. MCP wrapper version can be managed independently (SDK updates do not affect CLI)
3. Single command npx install: `"command": "npx", "args": ["-y", "@stayicon/drift-guard-mcp"]`
4. Anthropic MCP Registry recommends separate npm packages

---

## ADR-007: CLI-First Strategy — MCP Wrapper Deployment on Hold (2026-03-12)

### Context
v0.2.0 implementation of `drift-guard-mcp/` (4 MCP tools) complete. Reexamined MCP token overhead issue just before deployment.

### Research Findings (2026-03-12)
- **MCP token bloat** is the biggest developer community debate in 2026:
  - Anthropic official: 5 MCP servers = ~55K tokens (consumed before conversation starts)
  - Redis benchmark: 4 servers (167 tools) = ~60K tokens
  - Eric Holmes "MCP is dead" (HN #1): CLI `--help` = ~200 tokens vs MCP init = ~10,000+ tokens (50x difference)
  - Scalekit benchmark: same task MCP vs CLI = $55.20/month vs $3.20/month (17x cost)
- **"CLI over MCP" trend** forming (The New Stack, Zuplo, dev.to, r/AI_Agents)
- drift-guard target users = developers only — MCP not needed
- MCP support is irrelevant to OpenAI/Anthropic open source program eligibility

### Decision
**Hold** `@stayicon/drift-guard-mcp` npm publish. Code preserved in git but not deployed.

### Rationale
1. CLI = 0 token overhead — "zero-overhead" positioning is a differentiator amid MCP bloat debate
2. AI agents already run `npx drift-guard check` well — MCP wrapper unnecessary
3. Reduced maintenance cost (1 package vs 2 packages)
4. "Zero token overhead, pure CLI" message expected to resonate better on Show HN
5. Revisit when MCP lazy hydration/progressive disclosure becomes standard

---

## ADR-012: Windows Shell / Claude Code Invocation Protocol (2026-03-15)

### Context
When calling `@anthropic-ai/claude-code` via npx in a Windows environment, slash commands like `/plan`, `/fast` consistently cause shell parsing errors (slash-path confusion).

### Decision
Document slash command bypass rules in `claude-code-call-protocol.md` and global rules.
- Use numbered plain-text instructions (Numbered Steps) instead of slash commands.
- On error, immediately switch to direct instruction mode to prevent runtime hangs.

### Rationale
- Acknowledging and working around the slash handling limitations of the Windows bash environment is more productive.
- Without slash commands, Claude Code can still produce high-quality plans with clear instructions.
