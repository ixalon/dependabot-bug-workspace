# Dependabot Bug Demonstration

This repository demonstrates a bug in Dependabot where updating `@aws-sdk/client-dynamodb` in a workspace package removes nested optional peer dependencies, causing `npm ci` to fail.

## The Bug

When Dependabot creates a PR to update `@aws-sdk/client-dynamodb` from 3.943.0 to 3.983.0 in `packages/app/package.json`, it generates a `package-lock.json` that is missing these packages:
- `node_modules/@nestjs/schematics/node_modules/chokidar@3.6.0`
- `node_modules/@nestjs/schematics/node_modules/glob-parent@5.1.2`
- `node_modules/@nestjs/schematics/node_modules/readdirp@3.6.0`
- And their transitive dependencies

This causes `npm ci` to fail with: `npm error Missing: chokidar@3.6.0 from lock file`

## Repository Structure

This is an npm workspace with:
- **Root** (`package.json`): Contains devDependencies that are shared across the workspace
  - `@angular-devkit/schematics@^19.0.0` - requires `chokidar@^4.0.0` as optional peer
  - `@nestjs/schematics@^10.0.0` - bundles older `@angular-devkit/core@17.3.11` requiring `chokidar@^3.5.2` as optional peer
  - `chokidar@^4.0.0` - installed to satisfy the root-level peer dependency

- **packages/app** (`packages/app/package.json`): Contains the app dependencies
  - `@aws-sdk/client-dynamodb@^3.943.0` - the package Dependabot will update
  - `@nestjs/common@^10.0.0`
  - `@nestjs/core@^10.0.0`

## Root Cause

The issue occurs because:

1. **Conflicting Peer Dependencies**: Two versions of `@angular-devkit/core` exist:
   - `@angular-devkit/core@19.2.x` (root level) with peer dependency `chokidar@^4.0.0`
   - `@angular-devkit/core@17.3.11` (nested in `@nestjs/schematics`) with peer dependency `chokidar@^3.5.2`

2. **Nested Optional Peer**: `chokidar@3.6.0` is installed at `node_modules/@nestjs/schematics/node_modules/chokidar` and is marked in the lockfile as:
   ```json
   {
     "optional": true,
     "peer": true,
     "dev": true
   }
   ```

3. **Workspace Scenario**: Because `@aws-sdk/client-dynamodb` is NOT in the root `package.json` (only in `packages/app/package.json`), Dependabot's workflow hits the workspace cleanup code path in `npm_lockfile_updater.rb` (lines 279-303).

4. **Incorrect Pruning**: The workspace cleanup runs:
   ```bash
   npm install --package-lock-only --force --ignore-scripts
   ```
   This causes npm's resolution algorithm to remove the nested optional peer dependency `chokidar@3.6.0` because a newer version (`chokidar@4.0.3`) exists at the root level, even though both versions are needed.

5. **Invalid Lockfile**: The resulting lockfile is missing packages that `npm ci` expects, causing builds to fail.

## How to Reproduce

1. **Enable Dependabot**: This repository is configured to only update `@aws-sdk/client-dynamodb` in `packages/app`
2. **Wait for PR**: Dependabot will create a PR updating from 3.943.0 to 3.983.0
3. **Observe Failure**: The GitHub Action will run `npm ci` and fail with "Missing: chokidar@3.6.0"

## Expected Behavior

- `npm ci` should succeed on Dependabot PRs
- Optional peer dependencies should not be removed if they are referenced by nested packages
- The lockfile should remain valid after Dependabot updates

## Actual Behavior

- Dependabot generates an invalid lockfile
- `npm ci` fails with missing package errors
- The PR cannot be merged without manual intervention

## Technical Details

- **Node version**: 20
- **npm version**: Uses whatever Dependabot's updater container has (npm 11.x behavior differs from npm 10.x)
- **Issue type**: Lockfile generation bug in `npm_and_yarn/lib/dependabot/npm_and_yarn/file_updater/npm_lockfile_updater.rb`
- **Suspected code**: Lines 279-303 (workspace reset logic)

## Why This is a Bug

### Behavior Inconsistency

1. **`npm install` (with node_modules)**: Correctly maintains both chokidar versions
2. **`npm install --package-lock-only`** (Dependabot's workspace cleanup): Incorrectly removes the nested optional peer dependency
3. **`npm ci`**: Expects the packages that were originally in the lockfile

This creates a lockfile that cannot be installed with `npm ci`.

## Proposed Fixes for Dependabot Team

### Option 1: Preserve Optional Peer Dependencies

Before running the final cleanup, detect and preserve optional peer dependencies:

```ruby
unless dependencies_in_current_package_json
  File.write(T.must(package.json).name, previous_package_json)

  # Detect optional peer dependencies before final install
  before_packages = JSON.parse(File.read(lockfile_basename))["packages"]
  optional_peers = before_packages.select { |k, v| v["optional"] && v["peer"] }

  run_npm_install_lockfile_only([], has_optional_dependencies: false)

  # Verify optional peers weren't removed
  after_packages = JSON.parse(File.read(lockfile_basename))["packages"]
  removed_peers = optional_peers.keys - after_packages.keys

  if removed_peers.any?
    # Restore or error
  end
end
```

### Option 2: Use --legacy-peer-deps Flag

For workspace cleanup, use:
```bash
npm install --package-lock-only --force --ignore-scripts --legacy-peer-deps
```

### Option 3: Skip Workspace Cleanup for Peer Dependencies

Don't run the bare `npm install --package-lock-only` if the update involves packages with optional peer dependencies.

### Option 4: Validate Lockfile After Generation

After generating the lockfile, run validation:
```bash
npm ci --dry-run
```

If it fails with "Missing:", that's a signal the lockfile is broken.

## Additional Context

- Different npm versions handle optional peer dependencies differently (npm 10.x vs 11.x)
- The bug requires a specific combination:
  - Nested optional peer dependencies
  - Conflicting peer dependency versions
  - Workspace scenario (dependency not in root package.json)
  - Dependabot's workspace cleanup step

## Related Issue

See GitHub issue: https://github.com/dependabot/dependabot-core/issues/[issue-number]
