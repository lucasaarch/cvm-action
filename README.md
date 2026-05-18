# CVM Action - Crate Version Manager

A GitHub Action to automatically manage Rust crate versions with semantic versioning. This action integrates with [CVM (Crate Version Manager)](https://crates.io/crates/cvm_cli) to check for pending version changes and apply them via pull requests.

## 🚀 Quick Start

```yaml
name: Apply Version Bumps

on:
  workflow_dispatch:

jobs:
  version-bump:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: lucasaarch/cvm-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## ✨ Features

- 🎯 Automatic semantic versioning for Rust crates
- 🔄 Support for workspaces and multi-crate projects
- 📦 Staged version changes via `.cvm/changes/` files
- ⚡ CI-friendly with status checks and dry-run mode
- 🤖 Automatic PR creation with version bumps
- 🚀 Automated publishing to crates.io with tags and GitHub releases
- ✅ Preserves Cargo.toml formatting

## 📋 Complete CI/CD Pipeline Example

Here's a complete CI/CD pipeline that checks for pending version changes, applies them, and publishes to crates.io:

```yaml
name: Version Management & Publishing

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  check-and-apply:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    outputs:
      has-changes: ${{ steps.cvm.outputs.has-changes }}
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Check and apply version changes
        id: cvm
        uses: lucasaarch/cvm-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          pr-title: 'chore: apply version bumps'
          pr-labels: 'version-bump,automated'

  publish:
    runs-on: ubuntu-latest
    needs: check-and-apply
    if: needs.check-and-apply.outputs.has-changes == 'false'
    permissions:
      contents: write
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Publish crates
        uses: lucasaarch/cvm-action@v1
        with:
          command: 'publish'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

**How this pipeline works:**

### When there ARE pending changes:
1. ✅ Check for pending version changes in `.cvm/changes/`
2. 📝 Apply changes to `Cargo.toml` files
3. 🔀 Create a pull request with the version bumps
4. 🗑️ Remove applied change files
5. ⏸️ Skip publish step

### After merging the PR (no pending changes):
1. ✅ No pending changes detected
2. 🏷️ Create Git tags (e.g., `v1.2.3` or `crate-name-v1.2.3`)
3. 📦 Create GitHub releases
4. 🚀 Publish to crates.io

## 📖 How It Works

### 1. Create Version Changes Locally

Use the CVM CLI to create version change files:

```bash
# Install CVM
cargo install cvm_cli

# Interactive version bump
cvm

# This creates a file in .cvm/changes/
```

### 2. Commit Change Files

```bash
git add .cvm/changes/
git commit -m "chore: bump crate-a to v1.2.0"
git push
```

### 3. GitHub Action Applies Changes

The action will:
- Detect the change files
- Update `Cargo.toml` versions
- Create a PR with the changes
- Remove the change files

## 🎮 Usage Examples

### Dry Run (Preview Only)

Preview changes without applying them:

```yaml
- uses: lucasaarch/cvm-action@v1
  with:
    dry-run: 'true'
    create-pr: 'false'
```

### Status Check Only

Check for pending changes without applying:

```yaml
- name: Check for pending changes
  id: check
  uses: lucasaarch/cvm-action@v1
  with:
    command: 'status'
    create-pr: 'false'

- name: Fail if changes pending
  if: steps.check.outputs.has-changes == 'true'
  run: exit 1
```

### Publish to crates.io

Publish crates with tags and GitHub releases:

```yaml
- name: Publish crates
  uses: lucasaarch/cvm-action@v1
  with:
    command: 'publish'
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

### Publish with Custom Options

```yaml
# Publish without creating GitHub releases
- uses: lucasaarch/cvm-action@v1
  with:
    command: 'publish'
    publish-no-release: 'true'
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

# Publish without creating Git tags (only crates.io)
- uses: lucasaarch/cvm-action@v1
  with:
    command: 'publish'
    publish-no-tag: 'true'
    publish-no-release: 'true'
  env:
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

# Dry-run publish (preview what would be published)
- uses: lucasaarch/cvm-action@v1
  with:
    command: 'publish'
    dry-run: 'true'
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

### Custom PR Configuration

```yaml
- uses: lucasaarch/cvm-action@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    pr-title: 'chore(release): version bump'
    pr-labels: 'release,automated,dependencies'
```

### Manual Apply (No PR)

Apply changes without creating a PR:

```yaml
- uses: lucasaarch/cvm-action@v1
  with:
    create-pr: 'false'
```

Then commit manually in a subsequent step.

## ⚙️ Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `command` | Command to run: `status`, `apply`, or `check-and-apply` | `check-and-apply` |
| `dry-run` | Run in dry-run mode (preview only) | `false` |
| `create-pr` | Create a pull request with version changes | `true` |
| `pr-title` | Title for the pull request | `chore: apply version bumps` |
| `pr-labels` | Comma-separated labels for the PR | `version-bump,automated` |
| `publish-no-tag` | Skip creating Git tags when publishing | `false` |
| `publish-no-release` | Skip creating GitHub releases when publishing | `false` |
| `publish-allow-dirty` | Allow publishing with uncommitted changes | `false` |

## 📤 Outputs

| Output | Description |
|--------|-------------|
| `has-changes` | Whether there are pending version changes (`true`/`false`) |
| `applied` | Whether changes were applied (`true`/`false`) |
| `published` | Whether crates were published (`true`/`false`) |
| `version` | Unified release version (from `cvm info --version`, or when all published crates share the same version) |
| `current-version` | Current release version before apply/publish |
| `new-version` | Release version after applying pending changes |
| `crates` | JSON array of published crates with versions |

> Requires **cvm_cli 1.0.10+** for workspace `version.workspace = true` and `cvm info --version`.

## 📁 Change File Format

Change files are stored in `.cvm/changes/` as TOML files:

```toml
[update]
summary = "Add new authentication feature"
minor = ["my-auth-crate"]
patch = ["my-utils-crate"]
pre = false
```

## 🔧 Local CVM Commands

```bash
# Interactive version bump
cvm

# Check pending changes
cvm status

# Preview changes
cvm apply --dry-run

# Apply changes
cvm apply

# Start prerelease mode
cvm pre start canary

# Exit prerelease mode
cvm pre exit
```

## 🧪 Testing

Test the action locally:

```bash
./test-action-local.sh
```

See [TESTING.md](TESTING.md) for detailed testing instructions.

## 📝 Example Workflow Files

Check the [examples/](examples/) directory for:
- `basic.yml` - Simple weekly version bump
- `advanced.yml` - Full CI integration with checks

## 🔒 Permissions

### For applying version changes (creating PRs):

```yaml
permissions:
  contents: write
  pull-requests: write
```

### For publishing to crates.io:

```yaml
permissions:
  contents: write  # Required for creating tags and releases
```

### Environment Variables

**For creating PRs and GitHub releases:**
```yaml
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**For publishing to crates.io:**
```yaml
env:
  CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

Get your crates.io token from [crates.io/me](https://crates.io/me) and add it to your repository secrets.

## 🤝 Related Projects

- [CVM CLI](https://github.com/lucasaarch/cvm) - The core CVM tool
- [cvm_cli on crates.io](https://crates.io/crates/cvm_cli) - Install with `cargo install cvm_cli`

## 📄 License

MIT License - see the [LICENSE](LICENSE) file for details.

## 🐛 Issues & Contributing

- Report issues: [GitHub Issues](https://github.com/lucasaarch/cvm-action/issues)
- Contributions welcome! Please submit a Pull Request.

## 💡 Why Use CVM?

- **Staged Changes**: Version bumps are staged and reviewed before applying
- **Workspace Support**: Handles complex Rust workspaces with multiple crates
- **Prerelease Support**: Built-in support for prerelease versioning (canary, alpha, beta)
- **CI-Friendly**: Designed for automation with clear exit codes and dry-run mode
- **Format Preserving**: Maintains your Cargo.toml formatting and comments

---

**Made with ❤️ by lucasaarch**
