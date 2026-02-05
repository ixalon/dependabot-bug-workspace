# Dependabot Bug Demonstration

This repository demonstrates a bug in Dependabot where updating a dependency in an npm workspace removes nested optional peer dependencies, causing `npm ci` to fail with "Missing package from lock file" errors.

## The Bug

When Dependabot creates a PR to update `@aws-sdk/client-dynamodb` from 3.943.0 to 3.983.0, it generates a `package-lock.json` that is missing these packages:
- `node_modules/@nestjs/schematics/node_modules/chokidar@3.6.0` (optional peer dependency)
- `node_modules/@nestjs/schematics/node_modules/glob-parent@5.1.2`
- `node_modules/@nestjs/schematics/node_modules/readdirp@3.6.0`
- And their transitive dependencies

This causes `npm ci` to fail with:
```
npm error Missing: chokidar@3.6.0 from lock file
```

## Exact Requirements to Reproduce

**CRITICAL:** All of these conditions must be present simultaneously:

### 1. Must Use npm Workspaces
```json
{
  "workspaces": ["packages/*"]
}
```

### 2. Dependency Being Updated Must NOT Be in Root package.json
- ❌ **Wrong**: `@aws-sdk/client-dynamodb` in root `package.json`
- ✅ **Correct**: `@aws-sdk/client-dynamodb` in `packages/app/package.json` only

This is critical because Dependabot has different code paths:
- If dependency is in root package.json: Updates normally
- If dependency is NOT in root package.json: Triggers workspace cleanup code (lines 279-303 in npm_lockfile_updater.rb) **← This is where the bug happens**

### 3. Must Have Nested Optional Peer Dependencies with Version Conflict

You need packages that create this structure:
- **Root level**: Package requiring `library@^4.0.0` as optional peer → installs `library@4.x`
- **Nested**: Package bundling older version requiring `library@^3.0.0` as optional peer → installs `library@3.x` at nested location

In this demo:
- Root: `@angular-devkit/schematics@^19.0.0` requires `chokidar@^4.0.0` → installs `chokidar@4.0.3`
- Nested: `@nestjs/schematics@^10.0.0` bundles `@angular-devkit/core@17.3.11` requiring `chokidar@^3.5.2` → installs `chokidar@3.6.0` at `node_modules/@nestjs/schematics/node_modules/chokidar`

The nested `chokidar@3.6.0` is marked in the lockfile as:
```json
{
  "optional": true,
  "peer": true,
  "dev": true
}
```

### 4. Version Must Be Pinned in Lockfile

The lockfile must have the old version installed, not the latest:
- ❌ **Wrong**: Using `^3.943.0` in package.json causes `npm install` to install latest (3.983.0) immediately
- ✅ **Correct**: Pin to exact version `3.943.0` in package.json, then run `npm install` to lock it

### 5. Dependabot Config Must Scan Root Directory

For workspaces, Dependabot must scan from root:
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"  # ← Must be "/" not "/packages/app"
```

## Repository Structure

**Root package.json**:
```json
{
  "workspaces": ["packages/*"],
  "devDependencies": {
    "@angular-devkit/schematics": "^19.0.0",
    "@nestjs/schematics": "^10.0.0",
    "chokidar": "^4.0.0"
  }
}
```

**packages/app/package.json**:
```json
{
  "dependencies": {
    "@aws-sdk/client-dynamodb": "3.943.0",  // Note: Pinned, not ^3.943.0
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0"
  }
}
```

**Result**: Single `package-lock.json` at root with both chokidar versions

## Root Cause Analysis

### The Problematic Code Path

In `npm_lockfile_updater.rb` lines 279-303:

```ruby
unless dependencies_in_current_package_json
  # Dependencies NOT in root package.json - workspace scenario
  File.write(T.must(package_json).name, previous_package_json)

  # This is the problem ↓
  run_npm_install_lockfile_only([], has_optional_dependencies: false)
end
```

This runs:
```bash
npm install --package-lock-only --force --ignore-scripts
```

### What Goes Wrong

1. **Before update**: Lockfile has both `chokidar@4.0.3` (root) and `chokidar@3.6.0` (nested, optional peer)
2. **Dependabot updates**: `@aws-sdk/client-dynamodb` from 3.943.0 to 3.983.0
3. **Workspace cleanup runs**: `npm install --package-lock-only`
4. **npm sees**:
   - Optional peer dependency: `chokidar@3.6.0`
   - Newer version available: `chokidar@4.0.3`
   - Decision: Remove optional peer since newer version exists
5. **Result**: Lockfile missing `chokidar@3.6.0` and all its dependencies
6. **npm ci fails**: Expects `chokidar@3.6.0` but it's gone

### Why This is Wrong

**Behavior inconsistency**:
- `npm install` (with node_modules): ✅ Correctly maintains both versions
- `npm install --package-lock-only`: ❌ Incorrectly removes nested optional peer
- `npm ci`: ❌ Fails because packages are missing

The lockfile Dependabot generates cannot be installed with `npm ci`.

## What We Learned Creating This Demo

### Mistake 1: Non-Workspace Structure
**First attempt**: Put all dependencies in root `package.json`
- Result: Bug didn't reproduce because workspace cleanup code was never triggered
- Fix: Move `@aws-sdk/client-dynamodb` to `packages/app/package.json`

### Mistake 2: Using Caret Ranges
**Second attempt**: Used `^3.943.0` in package.json
- Result: `npm install` immediately installed 3.983.0, so Dependabot saw no update needed
- Fix: Pin to exact version `3.943.0` to lock it in the lockfile

### Mistake 3: Wrong Dependabot Directory
**Third attempt**: Used `directory: "/packages/app"`
- Result: Dependabot didn't detect the workspace properly
- Fix: Use `directory: "/"` for workspaces

### Mistake 4: Too Restrictive Allow Constraint
**Fourth attempt**: Used `allow: [dependency-name: "@aws-sdk/client-dynamodb"]`
- Result: Blocked Dependabot from creating any PRs (including security updates)
- Fix: Remove allow constraint entirely

## How to Reproduce

1. Clone this repository
2. Verify initial state:
   ```bash
   npm ls chokidar
   # Should show both 4.0.3 (root) and 3.6.0 (nested)
   ```
3. Wait for Dependabot PR or manually trigger from Insights → Dependency graph → Dependabot
4. Observe the PR's lockfile changes - `chokidar@3.6.0` will be removed
5. GitHub Action runs `npm ci` and fails with "Missing: chokidar@3.6.0"

## Expected vs Actual Behavior

**Expected**:
- ✅ `npm ci` succeeds on Dependabot PRs
- ✅ Optional peer dependencies are preserved when still needed by nested packages
- ✅ Lockfile remains valid

**Actual**:
- ❌ Dependabot generates invalid lockfile
- ❌ `npm ci` fails with "Missing package" errors
- ❌ PR cannot be merged without manual lockfile regeneration

## Proposed Fixes for Dependabot Team

### Option 1: Preserve Optional Peer Dependencies
```ruby
unless dependencies_in_current_package_json
  File.write(T.must(package_json).name, previous_package_json)

  # Snapshot before cleanup
  before_packages = JSON.parse(File.read(lockfile_basename))["packages"]
  optional_peers = before_packages.select { |k, v| v["optional"] && v["peer"] }

  run_npm_install_lockfile_only([], has_optional_dependencies: false)

  # Verify optional peers weren't removed
  after_packages = JSON.parse(File.read(lockfile_basename))["packages"]
  removed_peers = optional_peers.keys - after_packages.keys

  if removed_peers.any?
    raise "Optional peer dependencies removed: #{removed_peers}"
  end
end
```

### Option 2: Use --legacy-peer-deps
```bash
npm install --package-lock-only --force --ignore-scripts --legacy-peer-deps
```

### Option 3: Validate After Generation
```bash
npm ci --dry-run
```
If it fails with "Missing:", the lockfile is broken.

### Option 4: Skip Cleanup for Peer Dependencies
Don't run workspace cleanup if optional peer dependencies exist in the lockfile.

## Technical Details

- **npm versions**: Bug behavior differs between npm 10.x and 11.x
- **Node version**: 20 (but affects all versions)
- **Affected code**: `npm_and_yarn/lib/dependabot/npm_and_yarn/file_updater/npm_lockfile_updater.rb` lines 279-303
- **Impact**: Any npm workspace with nested optional peer dependencies

## Additional Notes

This is NOT:
- ❌ A configuration issue
- ❌ A private registry issue
- ❌ User error

This IS:
- ✅ A bug in Dependabot's lockfile generation for npm workspaces
- ✅ Reproducible with public npm packages
- ✅ Affects production builds (fails CI/CD)
