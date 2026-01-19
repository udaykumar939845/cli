# NPM Package Setup Guide

This guide explains how to set up and publish this Golang CLI as an npm package, similar to how Supabase CLI is distributed.

## Overview

This project uses a hybrid approach:
- **Go** for the CLI implementation (compiled to binaries)
- **NPM** for distribution and installation in JavaScript/TypeScript projects
- **GoReleaser** for building cross-platform binaries
- **Post-install script** that downloads the correct binary for the user's platform

## Prerequisites

1. **Go** (1.25+)
2. **Node.js** (v18+) and **npm** (v8+)
3. **GoReleaser** for building releases
4. **GitHub repository** for hosting releases

## Setup Steps

### 1. Update Project Information

Make sure you've updated the following files with your project details:

- `package.json` - Update `name`, `repository`, `homepage`, `bugs`, and `author`
- `.goreleaser.yml` - Update `project_name` and binary name
- `scripts/postinstall.js` - Already configured to read from `package.json`

### 2. Build and Release Go Binaries

#### Install GoReleaser

```bash
# macOS/Linux
brew install goreleaser/tap/goreleaser

# Or download from https://goreleaser.com/install
```

#### Create a GitHub Release

1. **Tag your release:**
   ```bash
   git tag -a v1.0.0 -m "Release v1.0.0"
   git push origin v1.0.0
   ```

2. **Build and release with GoReleaser:**
   ```bash
   goreleaser release --clean
   ```

   This will:
   - Build binaries for all platforms (darwin, linux, windows, amd64, arm64)
   - Create archives (`.tar.gz` files) named like `sopabase_darwin_amd64.tar.gz`
   - Generate checksums file (`sopabase_1.0.0_checksums.txt`)
   - Create a GitHub release with all artifacts

#### Manual Build (Alternative)

If you prefer to build manually:

```bash
# Build for current platform
go build -o bin/sopabase -trimpath -ldflags "-s -w" main.go

# Build for multiple platforms
GOOS=darwin GOARCH=amd64 go build -o bin/sopabase_darwin_amd64 -trimpath -ldflags "-s -w" main.go
GOOS=darwin GOARCH=arm64 go build -o bin/sopabase_darwin_arm64 -trimpath -ldflags "-s -w" main.go
GOOS=linux GOARCH=amd64 go build -o bin/sopabase_linux_amd64 -trimpath -ldflags "-s -w" main.go
GOOS=linux GOARCH=arm64 go build -o bin/sopabase_linux_arm64 -trimpath -ldflags "-s -w" main.go
GOOS=windows GOARCH=amd64 go build -o bin/sopabase_windows_amd64.exe -trimpath -ldflags "-s -w" main.go
GOOS=windows GOARCH=arm64 go build -o bin/sopabase_windows_arm64.exe -trimpath -ldflags "-s -w" main.go

# Create archives
tar -czf sopabase_darwin_amd64.tar.gz -C bin sopabase_darwin_amd64
tar -czf sopabase_darwin_arm64.tar.gz -C bin sopabase_darwin_arm64
tar -czf sopabase_linux_amd64.tar.gz -C bin sopabase_linux_amd64
tar -czf sopabase_linux_arm64.tar.gz -C bin sopabase_linux_arm64
tar -czf sopabase_windows_amd64.tar.gz -C bin sopabase_windows_amd64.exe
tar -czf sopabase_windows_arm64.tar.gz -C bin sopabase_windows_arm64.exe

# Generate checksums
shasum -a 256 *.tar.gz > sopabase_1.0.0_checksums.txt
```

### 3. Update package.json Version

Before publishing to npm, update the version in `package.json`:

```bash
npm version 1.0.0
```

Or manually edit `package.json` to set the version that matches your GitHub release tag.

### 4. Publish to NPM

#### Test Locally First

```bash
# Test the postinstall script locally
npm install

# Verify the binary was downloaded
./bin/sopabase --version
```

#### Publish to NPM

```bash
# Make sure you're logged in
npm login

# Publish (make sure version matches GitHub release)
npm publish
```

For scoped packages:
```bash
npm publish --access public
```

## Using in JavaScript/TypeScript Projects

### Installation

Users can install your package as a dev dependency:

```bash
npm install sopabase --save-dev
```

Or with yarn:
```bash
yarn add -D sopabase
```

### Usage

#### As a CLI Command

After installation, users can run:

```bash
# Using npx
npx sopabase <command>

# Or if installed globally (not recommended per postinstall script)
sopabase <command>
```

#### Programmatically in Node.js

Users can also execute the CLI programmatically:

```javascript
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function runSopabase() {
  try {
    const { stdout, stderr } = await execAsync('npx sopabase --version');
    console.log(stdout);
  } catch (error) {
    console.error(error);
  }
}
```

Or using the binary directly:

```javascript
import { spawn } from 'child_process';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const binPath = path.join(__dirname, '..', 'node_modules', 'sopabase', 'bin', 'sopabase');

const child = spawn(binPath, ['--version']);
child.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});
```

## Release Workflow

1. **Update code and commit changes**
2. **Update version in `package.json`** (e.g., `1.0.0`)
3. **Create and push Git tag:**
   ```bash
   git tag -a v1.0.0 -m "Release v1.0.0"
   git push origin v1.0.0
   ```
4. **Run GoReleaser to create GitHub release:**
   ```bash
   goreleaser release --clean
   ```
5. **Publish to npm:**
   ```bash
   npm publish
   ```

## Troubleshooting

### Binary Not Found After Install

- Check that the GitHub release exists and contains the correct binary for your platform
- Verify the version in `package.json` matches the GitHub release tag
- Check that the repository URL in `package.json` is correct

### Checksum Verification Fails

- Ensure the checksums file is uploaded to the GitHub release
- Verify the checksum file format matches what `postinstall.js` expects

### Unsupported Platform

The postinstall script currently supports:
- `darwin` (macOS) - amd64, arm64
- `linux` - amd64, arm64
- `win32` (Windows) - amd64, arm64

To add support for other platforms, update `ARCH_MAPPING` and `PLATFORM_MAPPING` in `scripts/postinstall.js`.

## CI/CD Integration

You can automate releases using GitHub Actions. Example workflow:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: '1.25'
      - uses: goreleaser/goreleaser-action@v4
        with:
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
  publish-npm:
    needs: release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
      - run: cd cli && npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Notes

- The `bin/` directory is gitignored as binaries are downloaded during `npm install`
- The postinstall script prevents global installation (by design)
- Binary downloads happen automatically when users run `npm install`
- Checksums are verified to ensure binary integrity
