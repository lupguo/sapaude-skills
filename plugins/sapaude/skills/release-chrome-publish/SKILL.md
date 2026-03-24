---
name: release-chrome-publish
description: |
  Chrome extension publishing workflow. Scaffolds root package.json + scripts/ publishing
  infrastructure (init mode), or executes the full release pipeline: esbuild prod build →
  semi-auto CHANGELOG → ZIP package → Chrome Web Store API push.
  Use when: publishing Chrome extension, creating release ZIP, pushing to Chrome
  Web Store, bumping extension version, setting up publish infrastructure.
triggers:
  - chrome-publish
  - chrome publish
  - publish extension
  - release extension
  - cws publish
---

# Chrome Extension Publishing Skill

A reusable publishing workflow for Chrome extensions using esbuild for production builds.
Source lives in `src/`, built artifacts go to `dist/dev/` (dev) and `dist/prod/` (prod).
Project-specific configuration lives in `scripts/config.js`.

## Project Layout (expected)

```
src/              ← all extension source files (edit these)
  manifest.json
  background.js / content.js / ...
  lib/            ← vendor libs + project libs
  styles/         ← theme CSS files
  ...
scripts/          ← build/publish Node.js scripts
  build.js        ← esbuild pipeline (dev/prod) — see below
  package-zip.js
  changelog.js
  publish.js
  config.js
dist/             ← generated, never edit (git-ignored)
  dev/            ← development build
  prod/           ← production build (.min.js / .min.css)
package.json      ← root devDependencies + npm scripts
```

## Invocation Modes

| Command | Action |
|---|---|
| `/chrome-publish init` | Scaffold root package.json + scripts/ in current project |
| `/chrome-publish release` | Full release pipeline (interactive) |
| `/chrome-publish package` | Prod build + ZIP only, no publish |
| `/chrome-publish readme` | Update README.md only |

---

## Mode: `init` — First-Time Setup

Use when the project does NOT yet have a root `package.json` or `scripts/` directory.

### Steps

1. **Check prerequisites**
   - Confirm working directory is a Chrome extension project (has `src/manifest.json`)
   - Check that `scripts/` and root `package.json` do not already exist

2. **Create root `package.json`**

```json
{
  "name": "chrome-extension",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "node scripts/build.js",
    "build": "node scripts/build.js --prod",
    "dev-watch": "node scripts/build.js --watch",
    "package": "npm run build && node scripts/package-zip.js",
    "changelog": "node scripts/changelog.js",
    "publish": "node scripts/publish.js",
    "readme": "node scripts/readme.js",
    "pre-release": "npm run package",
    "release": "npm run changelog && npm run package && npm run publish"
  },
  "devDependencies": {
    "archiver": "^7.0.1",
    "chrome-webstore-upload": "^3.1.0",
    "conventional-changelog": "^6.0.0",
    "dotenv": "^16.4.5",
    "esbuild": "^0.27.4"
  }
}
```

3. **Create `scripts/config.js`**

```js
// Publishing configuration for the Chrome extension.
// Paths are relative to the project root (one level above scripts/).

module.exports = {
  // Source manifest.json (version number lives here)
  manifestPath: '../src/manifest.json',

  // Production build directory (what gets packaged into ZIP)
  prodBuildDir: '../dist/prod',

  // ZIP output directory
  distDir: '../dist',
}
```

4. **Create `scripts/build.js`** — esbuild pipeline

```js
#!/usr/bin/env node
// Build pipeline: src/ → dist/dev/ (development) or dist/prod/ (production)
// Usage:
//   node scripts/build.js            → dev build (no minify, source maps)
//   node scripts/build.js --prod     → prod build (*.min.js / *.min.css)
//   node scripts/build.js --watch    → dev watch mode (rebuilds on change)

const esbuild = require('esbuild')
const fs      = require('fs')
const path    = require('path')

const isProd  = process.argv.includes('--prod')
const isWatch = process.argv.includes('--watch')
const mode    = isProd ? 'prod' : 'dev'

const projectRoot = path.resolve(__dirname, '..')
const srcDir      = path.join(projectRoot, 'src')
const outDir      = path.join(projectRoot, 'dist', mode)

// ── Entry points ─────────────────────────────────────────────────────────────
// Project-authored JS (minified in prod → *.min.js)
const JS_ENTRIES = [
  'background.js',
  'content.js',
  'lib/renderer.js',
  'lib/theme-manager.js',
  // add more project-authored JS files here
]

// Project-authored CSS (minified in prod → *.min.css)
const CSS_ENTRIES = [
  'styles/theme-github.css',
  'styles/theme-dark.css',
  // add more project-authored CSS files here
]

// ── Helpers ───────────────────────────────────────────────────────────────────

function resolveEntries(list) {
  return list
    .filter(f => fs.existsSync(path.join(srcDir, f)))
    .map(f => ({ rel: f, abs: path.join(srcDir, f) }))
}

// Copy all src/ files EXCEPT esbuild entry points (built by esbuild)
// and manifest.json (written separately with updated refs).
function copyStatic(entryAbsSet) {
  function walk(src, dest) {
    for (const entry of fs.readdirSync(src, { withFileTypes: true })) {
      const srcPath  = path.join(src, entry.name)
      const destPath = path.join(dest, entry.name)
      if (entry.name === 'manifest.json') continue
      if (entryAbsSet.has(srcPath)) continue
      if (entry.isDirectory()) {
        fs.mkdirSync(destPath, { recursive: true })
        walk(srcPath, destPath)
      } else {
        fs.mkdirSync(path.dirname(destPath), { recursive: true })
        fs.copyFileSync(srcPath, destPath)
      }
    }
  }
  walk(srcDir, outDir)
}

// Write manifest.json, updating JS/CSS references in prod.
function writeManifest(jsMap, cssMap) {
  const manifest = JSON.parse(
    fs.readFileSync(path.join(srcDir, 'manifest.json'), 'utf8')
  )
  if (isProd) {
    if (manifest.background?.service_worker) {
      manifest.background.service_worker =
        jsMap[manifest.background.service_worker] ?? manifest.background.service_worker
    }
    for (const cs of manifest.content_scripts ?? []) {
      if (cs.js)  cs.js  = cs.js.map(f  => jsMap[f]  ?? f)
      if (cs.css) cs.css = cs.css.map(f => cssMap[f] ?? f)
    }
  }
  fs.writeFileSync(
    path.join(outDir, 'manifest.json'),
    JSON.stringify(manifest, null, 2) + '\n'
  )
}

// Update script src / link href in HTML files (prod only).
// Also patches *.min.js files that contain CSS filename strings (e.g. theme-manager.js).
function updateRefs(jsMap, cssMap) {
  function patchFile(filePath, isJs) {
    let content = fs.readFileSync(filePath, 'utf8')
    let changed = false
    const map = isJs ? cssMap : { ...jsMap, ...cssMap }
    const fileDir = path.dirname(filePath)

    for (const [orig, min] of Object.entries(map)) {
      const origBase = path.basename(orig)
      const minBase  = path.basename(min)
      const origRel = path.relative(fileDir, path.join(outDir, orig)).split(path.sep).join('/')
      const minRel  = path.relative(fileDir, path.join(outDir, min)).split(path.sep).join('/')
      const updated = content
        .replaceAll(`"${origRel}"`, `"${minRel}"`)
        .replaceAll(`'${origRel}'`, `'${minRel}'`)
        .replaceAll(`"${origBase}"`, `"${minBase}"`)
        .replaceAll(`'${origBase}'`, `'${minBase}'`)
      if (updated !== content) { content = updated; changed = true }
    }
    if (changed) fs.writeFileSync(filePath, content)
  }

  function walk(dir) {
    for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
      const p = path.join(dir, entry.name)
      if (entry.isDirectory())           { walk(p); continue }
      if (entry.name.endsWith('.html'))   patchFile(p, false)
      if (entry.name.endsWith('.min.js')) patchFile(p, true)
    }
  }
  walk(outDir)
}

function dirSizeKb(dir) {
  let bytes = 0
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, entry.name)
    bytes += entry.isDirectory() ? Number(dirSizeKb(p)) * 1024 : fs.statSync(p).size
  }
  return (bytes / 1024).toFixed(0)
}

// ── Build ─────────────────────────────────────────────────────────────────────

async function build() {
  console.log(`\nBuilding [${mode}]...`)

  if (fs.existsSync(outDir)) fs.rmSync(outDir, { recursive: true, force: true })
  fs.mkdirSync(outDir, { recursive: true })

  const jsEntries   = resolveEntries(JS_ENTRIES)
  const cssEntries  = resolveEntries(CSS_ENTRIES)
  const entryAbsSet = new Set([...jsEntries, ...cssEntries].map(e => e.abs))

  copyStatic(entryAbsSet)

  await esbuild.build({
    entryPoints : [...jsEntries, ...cssEntries].map(e => e.abs),
    outbase     : srcDir,
    outdir      : outDir,
    bundle      : false,
    minify      : isProd,
    sourcemap   : isProd ? false : 'inline',
    entryNames  : isProd ? '[dir]/[name].min' : '[dir]/[name]',
    logLevel    : 'warning',
  })

  const jsMap  = {}
  const cssMap = {}
  if (isProd) {
    for (const { rel } of jsEntries)  jsMap[rel]  = rel.replace(/\.js$/,  '.min.js')
    for (const { rel } of cssEntries) cssMap[rel] = rel.replace(/\.css$/, '.min.css')
  }

  writeManifest(jsMap, cssMap)
  if (isProd) updateRefs(jsMap, cssMap)

  console.log(`✓ dist/${mode}/  (${dirSizeKb(outDir)} KB)\n`)
}

// ── Watch ─────────────────────────────────────────────────────────────────────

async function watch() {
  console.log('\nDev watch mode — building initial dist/dev/...')

  if (fs.existsSync(outDir)) fs.rmSync(outDir, { recursive: true, force: true })
  fs.mkdirSync(outDir, { recursive: true })

  const jsEntries   = resolveEntries(JS_ENTRIES)
  const cssEntries  = resolveEntries(CSS_ENTRIES)
  const entryAbsSet = new Set([...jsEntries, ...cssEntries].map(e => e.abs))

  copyStatic(entryAbsSet)
  writeManifest({}, {})

  const ctx = await esbuild.context({
    entryPoints : [...jsEntries, ...cssEntries].map(e => e.abs),
    outbase     : srcDir,
    outdir      : outDir,
    bundle      : false,
    sourcemap   : 'inline',
    logLevel    : 'info',
  })
  await ctx.watch()

  fs.watch(srcDir, { recursive: true }, (_event, filename) => {
    if (!filename) return
    const rel     = filename.split(path.sep).join('/')
    const srcPath = path.join(srcDir, rel)
    if (!fs.existsSync(srcPath) || entryAbsSet.has(srcPath) || rel === 'manifest.json') return
    const destPath = path.join(outDir, rel)
    fs.mkdirSync(path.dirname(destPath), { recursive: true })
    fs.copyFileSync(srcPath, destPath)
    console.log(`[copy] ${rel}`)
  })

  console.log('Watching src/ — reload Chrome extension after JS/CSS changes.')
  console.log('Chrome extension directory: dist/dev/')
}

// ── Entry ─────────────────────────────────────────────────────────────────────

if (isWatch) {
  watch().catch(e => { console.error(e); process.exit(1) })
} else {
  build().catch(e => { console.error(e); process.exit(1) })
}
```

5. **Create `scripts/package-zip.js`** — zips from `dist/prod/` (already clean)

```js
#!/usr/bin/env node
const fs       = require('fs')
const path     = require('path')
const archiver = require('archiver')
const config   = require('./config')

const prodDir  = path.resolve(__dirname, config.prodBuildDir)
const distDir  = path.resolve(__dirname, config.distDir)

if (!fs.existsSync(prodDir)) {
  console.error(`Production build not found: ${prodDir}\nRun 'npm run build' first.`)
  process.exit(1)
}

const manifest = JSON.parse(fs.readFileSync(path.join(prodDir, 'manifest.json'), 'utf8'))
const version  = manifest.version
const name     = manifest.name.toLowerCase().replace(/\s+/g, '-')

fs.mkdirSync(distDir, { recursive: true })
const zipPath = path.join(distDir, `${name}-${version}.zip`)

const output  = fs.createWriteStream(zipPath)
const archive = archiver('zip', { zlib: { level: 9 } })

output.on('close', () => {
  const kb = (archive.pointer() / 1024).toFixed(1)
  console.log(`\nPackaged: ${path.relative(path.resolve(__dirname, '..'), zipPath)} (${kb} KB)`)
})
archive.on('error', err => { throw err })
archive.on('entry', entry => { process.stdout.write(`  + ${entry.name}\n`) })
archive.pipe(output)
archive.glob('**/*', { cwd: prodDir, dot: false })
archive.finalize()
```

6. **Create `scripts/changelog.js`** — uses conventional-changelog Node API (no CLI)

```js
#!/usr/bin/env node
// Semi-automatic CHANGELOG.md generation from conventional commits.
// Flow: show version options → run conventional-changelog → open $EDITOR → write + bump + tag

const { execSync, spawnSync } = require('child_process')
const conventionalChangelog = require('conventional-changelog')
const fs = require('fs')
const path = require('path')
const readline = require('readline')
const config = require('./config')

const manifestPath = path.resolve(__dirname, config.manifestPath)
const manifest = JSON.parse(fs.readFileSync(manifestPath, 'utf8'))
const currentVersion = manifest.version
const [major, minor, patch] = currentVersion.split('.').map(Number)

const options = {
  patch: `${major}.${minor}.${patch + 1}`,
  minor: `${major}.${minor + 1}.0`,
  major: `${major + 1}.0.0`,
}

const rl = readline.createInterface({ input: process.stdin, output: process.stdout })

console.log(`\nCurrent version: ${currentVersion}`)
console.log(`  [1] patch → ${options.patch}`)
console.log(`  [2] minor → ${options.minor}`)
console.log(`  [3] major → ${options.major}`)
console.log(`  [4] custom`)

rl.question('\nChoose [1-4]: ', (choice) => {
  let newVersion
  if (choice === '1') newVersion = options.patch
  else if (choice === '2') newVersion = options.minor
  else if (choice === '3') newVersion = options.major
  else if (choice === '4') {
    rl.question('Enter version (x.y.z): ', (v) => {
      if (!/^\d+\.\d+\.\d+$/.test(v)) { console.error('Invalid version'); process.exit(1) }
      rl.close()
      runChangelog(v)
    })
    return
  } else {
    console.error('Invalid choice')
    process.exit(1)
  }
  rl.close()
  runChangelog(newVersion)
})

async function runChangelog(newVersion) {
  console.log(`\nGenerating changelog for v${newVersion}...`)

  // Generate changelog via Node API (no CLI dependency)
  const generated = await new Promise((resolve, reject) => {
    const chunks = []
    conventionalChangelog({ preset: 'angular', releaseCount: 1 }, null, null, null, null, {
      cwd: path.resolve(__dirname, '..'),
    })
      .on('data', chunk => chunks.push(chunk))
      .on('end', () => resolve(Buffer.concat(chunks).toString()))
      .on('error', reject)
  }).catch(e => { console.error('conventional-changelog failed:', e.message); process.exit(1) })

  const today = new Date().toISOString().split('T')[0]
  const entry = `## [${newVersion}] - ${today}\n\n${generated.trim()}\n\n`

  const editFile = path.join(require('os').tmpdir(), `changelog-edit-${Date.now()}.md`)
  const changelogPath = path.resolve(__dirname, '..', 'CHANGELOG.md')
  const existing = fs.existsSync(changelogPath) ? fs.readFileSync(changelogPath, 'utf8') : ''
  fs.writeFileSync(editFile, `${entry}${existing}`)

  console.log(`\nOpening editor for review (save and close to continue)...`)
  const editor = process.env.EDITOR || process.env.VISUAL || 'nano'
  const result = spawnSync(editor, [editFile], { stdio: 'inherit' })
  if (result.status !== 0) { console.error('Editor exited with error'); process.exit(1) }

  const reviewed = fs.readFileSync(editFile, 'utf8')
  fs.writeFileSync(changelogPath, reviewed)
  fs.unlinkSync(editFile)

  manifest.version = newVersion
  fs.writeFileSync(manifestPath, JSON.stringify(manifest, null, 2) + '\n')

  execSync(`git add CHANGELOG.md "${manifestPath}"`)
  execSync(`git commit -m "chore(release): v${newVersion}"`)
  execSync(`git tag v${newVersion}`)

  console.log(`\n✓ CHANGELOG.md updated`)
  console.log(`✓ manifest.json version → ${newVersion}`)
  console.log(`✓ git tag v${newVersion} created`)
  console.log(`\nRun 'git push && git push --tags' when ready.`)
}
```

7. **Create `scripts/publish.js`**

```js
#!/usr/bin/env node
require('dotenv').config({ path: require('path').resolve(__dirname, '../.env') })
const fs = require('fs')
const path = require('path')
const config = require('./config')

const { EXTENSION_ID, CLIENT_ID, CLIENT_SECRET, REFRESH_TOKEN } = process.env
if (!EXTENSION_ID || !CLIENT_ID || !CLIENT_SECRET || !REFRESH_TOKEN) {
  console.error('Missing .env credentials. Copy .env.example to .env and fill in values.')
  process.exit(1)
}

const prodDir  = path.resolve(__dirname, config.prodBuildDir)
const manifest = JSON.parse(fs.readFileSync(path.join(prodDir, 'manifest.json'), 'utf8'))
const version  = manifest.version
const name     = manifest.name.toLowerCase().replace(/\s+/g, '-')
const zipPath  = path.join(path.resolve(__dirname, config.distDir), `${name}-${version}.zip`)

if (!fs.existsSync(zipPath)) {
  console.error(`ZIP not found: ${zipPath}\nRun 'npm run package' first.`)
  process.exit(1)
}

async function run() {
  const webStore = require('chrome-webstore-upload')({
    extensionId: EXTENSION_ID, clientId: CLIENT_ID,
    clientSecret: CLIENT_SECRET, refreshToken: REFRESH_TOKEN,
  })
  console.log(`\nUploading ${path.basename(zipPath)} to Chrome Web Store...`)
  const uploadResult = await webStore.uploadExisting(fs.createReadStream(zipPath))
  if (uploadResult.uploadState !== 'SUCCESS') {
    console.error('Upload failed:', JSON.stringify(uploadResult, null, 2)); process.exit(1)
  }
  console.log(`✓ Upload successful (v${version})`)
  console.log('Publishing...')
  const publishResult = await webStore.publish()
  console.log('Publish result:', JSON.stringify(publishResult, null, 2))
  if (publishResult.status?.includes('OK')) {
    console.log(`\n✓ Published v${version} to Chrome Web Store!`)
  } else {
    console.error('Publish may have issues — check Developer Dashboard'); process.exit(1)
  }
}
run().catch(e => { console.error(e); process.exit(1) })
```

8. **Create `scripts/readme.js`**

```js
#!/usr/bin/env node
const { spawnSync } = require('child_process')
const path = require('path')
const readmePath = path.resolve(__dirname, '..', 'README.md')
const editor = process.env.EDITOR || process.env.VISUAL || 'nano'
console.log(`Opening README.md in ${editor}...`)
const result = spawnSync(editor, [readmePath], { stdio: 'inherit' })
if (result.status === 0) {
  console.log('\n✓ README.md updated. Commit with:')
  console.log('  git add README.md && git commit -m "docs: update README"')
} else { console.error('Editor exited with error'); process.exit(1) }
```

9. **Create `Makefile`** (project root, TAB-indented recipes)

```makefile
# Chrome Extension — Development & Publishing
# Run 'npm install' once after cloning.
# Load dist/dev/ in Chrome for development.

.PHONY: dev dev-watch build package changelog publish release readme pre-release

dev:
	@npm run dev

dev-watch:
	@npm run dev-watch

build:
	@npm run build

package:
	@npm run package

changelog:
	@npm run changelog

publish:
	@npm run publish

pre-release:
	@npm run pre-release

release:
	@npm run release

readme:
	@npm run readme
```

10. **Create `.env.example`** (project root)

```
# Chrome Web Store OAuth2 credentials
# See: https://developer.chrome.com/docs/webstore/using-api/
# Copy this file to .env and fill in your values.

EXTENSION_ID=
CLIENT_ID=
CLIENT_SECRET=
REFRESH_TOKEN=
```

11. **Run setup**

```bash
npm install
cp .env.example .env
# Edit .env with Chrome Web Store OAuth credentials
```

---

## Mode: `release` — Full Release Pipeline

Use when the project already has root `package.json` and `scripts/` from init.

### Steps

1. **Confirm clean git state**
   ```bash
   git status
   ```
   All changes should be committed before release. If not, commit or stash first.

2. **Run changelog** (interactive: choose version, review in editor)
   ```bash
   npm run changelog
   ```

3. **Build and package**
   ```bash
   npm run package
   ```
   This runs `npm run build` (→ `dist/prod/`) then `node scripts/package-zip.js` (→ `dist/*.zip`).

4. **Verify ZIP before publishing**
   ```bash
   unzip -l dist/*.zip | head -40
   ```
   Confirm all `.min.js`/`.min.css` files are present. Size should be under 10MB.

5. **Publish to Chrome Web Store**
   ```bash
   npm run publish
   ```
   Requires `.env` with valid OAuth credentials (see OAuth Setup below).

6. **Push git tags**
   ```bash
   git push && git push --tags
   ```

---

## Mode: `package` — Build ZIP Only

```bash
npm run package
unzip -l dist/*.zip
```

Use to inspect the ZIP before deciding to publish.

---

## Mode: `readme` — Update README Only

```bash
npm run readme
```

Opens `README.md` in `$EDITOR`. After saving, commit manually:
```bash
git add README.md && git commit -m "docs: update README"
```

---

## OAuth Setup (one-time, per developer)

To enable `npm run publish`, you need Google OAuth2 credentials:

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project → Enable "Chrome Web Store API"
3. Create OAuth 2.0 credentials (Desktop app type)
4. Get refresh token using the OAuth2 flow
5. Fill in `.env`:
   ```
   EXTENSION_ID=<your extension ID from Chrome Developer Dashboard>
   CLIENT_ID=<from Google Cloud Console>
   CLIENT_SECRET=<from Google Cloud Console>
   REFRESH_TOKEN=<from OAuth2 flow>
   ```

Reference: https://developer.chrome.com/docs/webstore/using-api/

---

## Key Conventions

- `node_modules/` is **gitignored** — run `npm install` at project root after cloning
- `.env` is **gitignored** — never commit credentials
- **Only edit files in `src/`** — `dist/` is generated and gitignored
- Vendor libs (`src/lib/*.min.js`) are **never re-minified** by esbuild (copied as-is via `copyStatic`)
- esbuild uses `bundle: false` to preserve `window.*` global exports required by MV3 content scripts
- `scripts/config.js` is the only file that needs editing per project
- Conventional Commits format (`feat:`, `fix:`, `chore:`) is required for quality changelog generation
- `dist/dev/` = development build (load in Chrome); `dist/prod/` = production build (what gets zipped)
