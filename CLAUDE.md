# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mermaid is a JavaScript-based diagramming tool that converts Markdown-inspired text into diagrams (flowcharts, sequence diagrams, class diagrams, etc.). This is a **pnpm monorepo** with 9 packages, built using esbuild and TypeScript.

## Essential Commands

### Development
```bash
pnpm install              # Install all dependencies (run after clone)
pnpm build                # Build all packages (esbuild + TypeScript types)
pnpm dev                  # Start dev server at http://localhost:9000
pnpm test                 # Run linting + unit tests (Vitest)
pnpm test:watch           # Run tests in watch mode
pnpm lint                 # Lint code (ESLint + Prettier check)
pnpm lint:fix             # Auto-fix linting issues
```

### Testing
```bash
pnpm test:coverage        # Run tests with coverage
pnpm cypress:open         # Open Cypress for E2E tests (requires dev server running)
pnpm e2e                  # Run E2E tests in headless mode
pnpm ci                   # Run tests in CI mode
```

### Building Specific Packages
```bash
pnpm build:esbuild        # Build packages only (no types)
pnpm build:types          # Generate TypeScript definitions only
pnpm build:mermaid        # Build mermaid package only
```

### Documentation
```bash
pnpm --filter mermaid run docs:dev    # Start docs site at http://localhost:3333
cd packages/mermaid && pnpm docs:dev  # Alternative way to run docs
```

### Docker Development
If using Docker, prefix commands with `./run`:
```bash
./run pnpm install
./run pnpm dev
./run pnpm test
```

## Architecture Overview

### Monorepo Structure

```
/packages/
├── mermaid/                    # Main package - core library
├── parser/                     # Langium-based parser (@mermaid-js/parser)
├── mermaid-example-diagram/    # Example external diagram
├── mermaid-zenuml/             # ZenUML sequence diagrams
├── mermaid-layout-elk/         # ELK layout engine
├── mermaid-layout-tidy-tree/   # Tidy tree layout
├── tiny/                       # Minimal build (no large features)
└── examples/                   # Usage examples
```

### Core Package Structure

```
packages/mermaid/src/
├── mermaid.ts              # Main entry point (public API)
├── mermaidAPI.ts           # Internal API (rendering, security)
├── Diagram.ts              # Diagram class (orchestration)
├── config.ts               # Configuration management
├── diagrams/               # All diagram implementations (25+ types)
│   ├── flowchart/         # Each diagram has: detector, db, renderer, parser, styles
│   ├── sequence/
│   └── .../
├── diagram-api/           # Diagram system infrastructure
│   ├── diagramAPI.ts     # Registration
│   ├── detectType.ts     # Detection
│   └── diagram-orchestration.ts  # Lazy loading
├── rendering-util/       # Shared rendering code (shapes, layout)
├── dagre-wrapper/        # Dagre layout wrapper
├── themes/               # Color themes
└── docs/                 # Documentation source (DO NOT edit /docs directly!)
```

### Diagram Lifecycle

1. **Text Input** → Preprocessing (extract frontmatter, directives)
2. **Type Detection** → Run detector functions until one matches
3. **Lazy Loading** → Load diagram code if not already loaded
4. **Parsing** → Parser populates DB with diagram data
5. **Rendering** → Renderer draws SVG using DB data
6. **Output** → Return SVG with styles and interactivity

### Diagram Definition Pattern

Each diagram type follows this structure:

```
/packages/mermaid/src/diagrams/<diagram-type>/
├── <type>Detector.ts    # Detects diagram type from text
├── <type>Diagram.ts     # Exports diagram definition
├── <type>Db.ts          # State management (DB pattern)
├── <type>Renderer.ts    # SVG rendering logic
├── styles.ts            # Diagram-specific CSS
├── parser/              # JISON or Langium parser
└── types.ts             # TypeScript definitions
```

**DiagramDefinition Interface:**
- `parser` - Parses text and populates DB
- `db` - Stores diagram state/data
- `renderer` - Draws diagram to SVG
- `styles` - CSS styles
- `init()` - Initialize with config

### Parser System

Two approaches exist:

**JISON (Legacy):**
- Grammar-based parser (`.jison` files)
- Compiled to JavaScript at build time
- Used by flowchart, gantt, class diagrams, etc.

**Langium (Modern):**
- Located in `/packages/parser/src/language/`
- Modern DSL framework
- Run `langium generate` to regenerate
- Used by architecture, gitGraph, packet, pie, radar, treemap

### Key Architectural Patterns

1. **Lazy Loading**: Diagrams registered with detector + loader, only loaded when detected
2. **Database Pattern**: Each diagram has a DB object (state management)
3. **Plugin Architecture**: External diagrams via `registerExternalDiagrams()`
4. **Factory Pattern**: `Diagram.fromText()` creates diagram instances
5. **Dependency Injection**: Common utilities via `injectUtils()`

## Development Guidelines

### Working on Diagram Types

1. Make changes in `/packages/mermaid/src/diagrams/<type>/`
2. Test locally using dev server: create/edit files in `/demos/dev/`
3. Access at `http://localhost:9000/dev/your-file.html`
4. Dev server auto-reloads on changes

### Adding a New Diagram

1. Create directory: `/packages/mermaid/src/diagrams/<type>/`
2. Implement: detector, diagram definition, DB, renderer, parser, styles
3. Register in `diagram-orchestration.ts`
4. Add unit tests: `<type>.spec.ts`
5. Add E2E tests in `/cypress/integration/`
6. Update documentation in `/packages/mermaid/src/docs/`

### Testing Requirements

**Unit Tests (Mandatory):**
- Use Vitest
- Co-locate with source: `*.spec.ts`
- For DOM testing: use `jsdomIt` helper (from `tests/util.ts`)
- Note: Layout tests require E2E (JSDOM has no rendering engine)

**E2E Tests (For Rendering):**
- Use Cypress with `imgSnapshotTest()` helper
- Visual regression testing (snapshots compared in CI)
- Required for any visual/layout changes

### Branch Naming Convention

```
[feature | bug | chore | docs]/[issue-number]_[short-description]
```

Examples:
- `feature/2945_state-diagram-new-arrow-florbs`
- `bug/1123_fix_random_ugly_red_text`
- `docs/2910_update-contributing-guidelines`

### Documentation

- **Source**: `/packages/mermaid/src/docs/` (edit here)
- **Published**: `/docs/` (auto-generated, DO NOT edit manually)
- Run docs locally: `pnpm --filter mermaid run docs:dev`
- Use markdown with special blocks: `note`, `tip`, `warning`, `danger`
- For new features: Add `(v<MERMAID_RELEASE_VERSION>+)` in title

### Git Workflow

- Main development branch: `develop`
- Base all work on `develop`
- Create feature branch → make changes → submit PR
- PR description should include: `Resolves #<issue-number>`

## Build System

### esbuild Configuration

Located in `/.esbuild/build.ts`:

**Output Formats:**
- ESM: `mermaid.core.mjs` (tree-shakeable)
- IIFE: `mermaid.js` (browser global)
- Minified: `mermaid.min.js`
- Tiny: `mermaid.tiny.min.js` (essential diagrams only)

**Build Process:**
1. Generate Langium parsers
2. Build parser package (dependency)
3. Build other packages with esbuild
4. Generate TypeScript definitions from `tsconfig.json`

### Custom Build Plugins

- JISON plugin: `.vite/jisonPlugin.js` (compiles `.jison` → JS)
- JSON Schema plugin: `.vite/jsonSchemaPlugin.js` (generates types)

## Configuration System

**Hierarchy (lowest to highest priority):**
1. `defaultConfig` (built-in)
2. Site config (`mermaid.initialize()`)
3. Directive config (`%%{init: {...}}%%`)
4. Frontmatter (YAML)

## Key Dependencies

- **D3.js** (v7): SVG manipulation
- **dagre-d3-es**: Graph layout
- **elkjs**: Eclipse Layout Kernel (external package)
- **roughjs**: Hand-drawn style rendering
- **DOMPurify**: Security (sanitize HTML/SVG)
- **Langium**: Modern parser framework

## Performance Considerations

- Full build: ~1MB (all diagrams)
- Core ESM: ~600KB (tree-shakeable)
- Tiny build: ~300KB (essential only)
- Lazy loading reduces initial bundle significantly

## Security Notes

- DOMPurify sanitizes all output
- Sandbox mode: renders in iframe for untrusted content
- Security issues: email security@mermaid.live

## Common Pitfalls

1. **Don't edit `/docs/` directly** - edit `/packages/mermaid/src/docs/` instead
2. **JSDOM limitations** - layout tests require E2E (no rendering engine in JSDOM)
3. **Parser changes** - regenerate Langium parsers with `langium generate`
4. **Type generation** - types auto-generated from JSON schema, don't edit manually
5. **Monorepo deps** - parser package must build before mermaid package

## Useful File Locations

- Main API: `packages/mermaid/src/mermaid.ts`
- Diagram registration: `packages/mermaid/src/diagram-api/diagram-orchestration.ts`
- Config types: `packages/mermaid/src/config.type.ts`
- Test utilities: `tests/util.ts`
- Build script: `.esbuild/build.ts`
- TypeScript config: `tsconfig.json`
- ESLint config: `eslint.config.js`
