# Node.js Prepare Action

A comprehensive GitHub composite action that streamlines Node.js project setup by handling repository checkout, Node.js runtime configuration, dependency caching, and installation in a single reusable step.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

This action combines common Node.js setup tasks into a single, optimized workflow step. It's designed to reduce boilerplate in your CI/CD pipelines while providing intelligent caching strategies for both development and production environments.

### Features

- 🚀 **All-in-one setup**: Checkout, Node.js setup, caching, and installation in one action
- 📦 **Smart caching**: Separate cache strategies for development and production dependencies
- ⚡ **Performance optimized**: Skips installation if dependencies are already cached
- 🔧 **Flexible configuration**: Optional repository checkout and environment modes
- 📌 **Version management**: Automatic Node.js version detection from `.nvmrc` file
- 🛡️ **Production ready**: Support for production-only dependency installation

## Usage

### Basic Usage

Add this action to your workflow to set up a Node.js project with default settings:

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare Node.js project
        uses: mkntz/actions-nodejs-prepare@v0
      
      - name: Run tests
        run: npm test
```

### Advanced Usage

#### Production Mode

For deployment workflows, use production mode to install only production dependencies:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare Node.js project (production)
        uses: mkntz/actions-nodejs-prepare@v0
        with:
          production: true
      
      - name: Build
        run: npm run build
      
      - name: Deploy
        run: npm run deploy
```

#### Skip Checkout

If you've already checked out the repository or need custom checkout options:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout with submodules
        uses: actions/checkout@v6
        with:
          submodules: recursive
      
      - name: Prepare Node.js project
        uses: mkntz/actions-nodejs-prepare@v0
        with:
          checkout: false
      
      - name: Build
        run: npm run build
```

#### Matrix Builds

Perfect for testing across multiple Node.js versions:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v6
      
      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      
      - name: Install dependencies
        run: npm ci
      
      # Or use this action with checkout disabled
      - name: Prepare Node.js project
        uses: mkntz/actions-nodejs-prepare@v0
        with:
          checkout: false
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `checkout` | Whether to checkout the repository using `actions/checkout@v6` | No | `true` |
| `production` | Whether to install only production dependencies (uses `npm ci --omit=dev`) | No | `false` |

## Behavior

### What This Action Does

1. **Checkout** (if `checkout: true`):
   - Clones the repository using `actions/checkout@v6`

2. **Node.js Setup**:
   - Configures Node.js runtime using the version specified in `.nvmrc`
   - Enables npm cache through `actions/setup-node@v6`

3. **Dependency Caching**:
   - **Development mode** (`production: false`):
     - Cache key: `${{ runner.os }}-node-dev-${{ hashFiles('**/package-lock.json') }}`
     - Caches all `node_modules` directories
   - **Production mode** (`production: true`):
     - Cache key: `${{ runner.os }}-node-prod-${{ hashFiles('**/package-lock.json') }}`
     - Caches production-only `node_modules`

4. **Dependency Installation**:
   - Only runs if `node_modules` doesn't exist (cache miss)
   - **Development mode**: Runs `npm ci --ignore-scripts`
   - **Production mode**: Runs `npm ci --omit=dev --ignore-scripts`

### Cache Strategy

The action uses separate cache keys for development and production environments to ensure:

- Development builds don't accidentally use production-only dependencies
- Production builds remain lean without dev dependencies
- Cache hits are maximized for each environment type

Cache is automatically invalidated when `package-lock.json` changes.

## Requirements

### Repository Requirements

Your repository must include:

- **`.nvmrc` file**: Specifies the Node.js version to use

  ```txt
  20.10.0
  ```

  or

  ```txt
  20
  ```

- **`package-lock.json` file**: Required for `npm ci` and cache key generation

### GitHub Actions Permissions

No special permissions required beyond default repository access.

## How It Works

This is a **composite action** that orchestrates several official GitHub Actions:

- [`actions/checkout@v6`](https://github.com/actions/checkout) - Repository checkout
- [`actions/setup-node@v6`](https://github.com/actions/setup-node) - Node.js runtime setup
- [`actions/cache@v4`](https://github.com/actions/cache) - Dependency caching

### Performance Benefits

- **First run**: ~2-3 minutes (full dependency installation)
- **Cached runs**: ~10-30 seconds (cache restore only)
- **Bandwidth savings**: Significant reduction in npm registry requests

## Examples

### Monorepo Setup

```yaml
jobs:
  test-packages:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare monorepo
        uses: mkntz/actions-nodejs-prepare@v0
      
      - name: Test all packages
        run: npm run test:all
      
      - name: Build all packages
        run: npm run build:all
```

### Conditional Production Build

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare Node.js
        uses: mkntz/actions-nodejs-prepare@v0
        with:
          production: ${{ github.ref == 'refs/heads/main' }}
      
      - name: Build
        run: npm run build
```

### Multi-step Workflow

```yaml
jobs:
  lint-test-build:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare Node.js project
        uses: mkntz/actions-nodejs-prepare@v0
      
      - name: Lint
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

## Comparison with Manual Setup

### Before (Manual Setup)

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v6
  
  - name: Set up Node.js
    uses: actions/setup-node@v6
    with:
      node-version-file: .nvmrc
      cache: npm
  
  - name: Cache node modules
    uses: actions/cache@v4
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
  
  - name: Install dependencies
    run: npm ci
```

### After (Using This Action)

```yaml
steps:
  - name: Prepare Node.js project
    uses: mkntz/actions-nodejs-prepare@v0
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

**Makan Taghizadeh** - [@mkntz](https://github.com/mkntz)

## Support

If you encounter any issues or have questions:

- Open an issue on [GitHub Issues](https://github.com/mkntz/actions-nodejs-prepare/issues)
- Check existing issues for solutions
