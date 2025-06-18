# Automating Rust Crate Publishing with GitHub Actions: A Production-Ready Approach

Publishing Rust crates manually is tedious and error-prone.
After maintaining a multi-crate workspace for Solana programs, I built a GitHub Actions workflow that automates the entire process—from verified builds to crates.io publishing to GitHub releases.
Here's what I learned and how you can implement something similar.

## The Problem

When you're maintaining multiple related crates in a workspace, manual publishing becomes a nightmare:

- **Version management chaos** - Keeping track of which crate needs which version bump
- **Testing gaps** - Forgetting to run full test suites before publishing
- **Release inconsistency** - Missing changelog updates or GitHub releases
- **Human error** - Publishing the wrong version or forgetting dependencies

## The Solution: A Three-Stage Pipeline

I designed a workflow with three distinct jobs that must pass sequentially:

### 1. Verified Build (`verified_build`)
```yaml
- run: docker pull --platform linux/amd64 solanafoundation/solana-verifiable-build:2.1.11
- run: solana-verify build --library-name jito_restaking_program --base-image solanafoundation/solana-verifiable-build:2.1.11
```

This stage builds our Solana programs using the official verifiable build image and uploads the artifacts.
The key insight here is that verified builds should happen *before* any publishing—if your program can't build reproducibly, you shouldn't publish anything.

### 2. Comprehensive Testing (`test_sbf`)
```yaml
- name: Download restaking program
  uses: actions/download-artifact@v4
  with:
    name: jito_restaking_program.so
    path: target/sbf-solana-solana/release/
```

The testing job downloads the verified artifacts and runs the full test suite against them.
This ensures we're testing the exact binaries that were built in the previous step—not just whatever happens to compile locally.

### 3. Smart Publishing (`publish`)

The publishing stage is where the magic happens. It uses `cargo-release` to handle version bumping, tagging, and publishing, but with several production-ready features:

**Flexible version control:**
```yaml
inputs:
  level:
    description: Version increment level
    type: choice
    options: [patch, minor, major]
```

**Dry-run capability:**
```yaml
if [ "${{ inputs.dry_run }}" == "true" ]; then
  cargo release ${{ inputs.level }} --no-confirm --no-push
else
  cargo release ${{ inputs.level }} --no-confirm -x
fi
```

**Automatic changelog generation:**
```yaml
- name: Generate a changelog
  uses: metcalfc/changelog-generator@v4.1.0
  with:
    includePattern: ".*/${{ inputs.package_path }}/.*"
```

## Key Implementation Details

### Multi-Crate Workspace Support

The workflow accepts a `package_path` input that specifies which crate to publish:

```yaml
package_path:
  type: choice
  options:
    - account_traits_derive
    - clients/rust/common
    - clients/rust/restaking_client
    # ... more crates
```

This approach lets you publish individual crates from a monorepo without affecting others.

### Version Tracking and Tagging

One tricky aspect was extracting version information for proper Git tagging:

```bash
# Get current version before update
OLD_VERSION=$(grep -m1 'version =' Cargo.toml | cut -d '"' -f2)

# Run cargo-release
cargo release ${{ inputs.level }} --no-confirm -x

# Get new version after update  
NEW_VERSION=$(grep -m1 'version =' Cargo.toml | cut -d '"' -f2)

# Create meaningful tag
echo "new_git_tag=${{ steps.extract_name.outputs.crate_name }}-v${NEW_VERSION}" >> $GITHUB_OUTPUT
```

### Conditional GitHub Releases

Not every crate publish needs a GitHub release, so I made it optional:

```yaml
- name: Create GitHub release
  if: github.event.inputs.create_release == 'true' && github.event.inputs.dry_run != 'true'
  uses: ncipollo/release-action@v1
```

## Lessons Learned

### 1. Always Use Dry-Run First
The dry-run feature saved me countless times. Publishing is irreversible, so being able to test the entire workflow without actually publishing is crucial.

### 2. Artifact Dependencies Matter
Initially, I had the test job trying to build programs independently. This created a race condition where tests might pass locally but fail in CI due to different build environments. Using artifacts ensures consistency.

### 3. Git Configuration is Critical
GitHub Actions runs need proper Git configuration for cargo-release to work:

```yaml
- name: Set Git Author
  run: |
    git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
    git config --global user.name "github-actions[bot]"
```

### 4. Permissions Must Be Explicit
Don't forget the `contents: write` permission for creating releases:

```yaml
jobs:
  publish:
    permissions:
      contents: write
```

## Setup Requirements

Before you can use this workflow, you'll need to set up a few secrets in your GitHub repository:

### Getting Your Cargo Registry Token

1. Log into [crates.io](https://crates.io/) and go to your Account Settings
2. Create a new API token with a descriptive name (e.g., "GitHub Actions Publishing")
3. Copy the token - you'll only see it once!
4. Add it to GitHub Secrets:
    - Go to your repository → Settings → Secrets and variables → Actions
    - Click "New repository secret"
    - Name: CARGO_REGISTRY_TOKEN
    - Value: Your token from crates.io

## Running the Workflow

The workflow is triggered manually with `workflow_dispatch`, giving you full control:

1. **Choose your crate** from the dropdown
2. **Select version increment** (patch/minor/major)  
3. **Enable dry-run** for testing
4. **Decide on GitHub release** creation

For a typical patch release, I run:
- First with `dry_run: true` to verify everything works
- Then with `dry_run: false` to actually publish

## What's Next?

This workflow handles the basics well, but there are areas for improvement:

- **Multi-crate publishing** - Currently, the workflow only accepts a single crate at a time. When you need to publish multiple related crates (especially with dependency chains), running the workflow multiple times becomes tedious. I'm planning to add support for selecting multiple crates or even publishing entire dependency trees in the correct order automatically.
- **Dependency awareness** - Automatically publishing dependent crates in the right order
- **Breaking change detection** - Using tools like `cargo-semver-checks`
- **Integration testing** - Testing published crates in downstream projects

## Conclusion

Automating crate publishing isn't just about convenience—it's about reliability and consistency. This workflow has eliminated publishing errors and made releases predictable across our entire team.

The key is building confidence through staging: verified builds prove reproducibility, comprehensive testing proves correctness, and dry-run capabilities prove the automation works before you commit.

Whether you're publishing Solana programs or any other Rust crates, the patterns here should give you a solid foundation for your own automation.

---

*The complete workflow file is available [here](https://github.com/jito-foundation/restaking/blob/master/.github/workflows/publish-crate.yaml).
