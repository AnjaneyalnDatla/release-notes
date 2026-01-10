# Create/Update Release Action

A composite GitHub Action that automatically creates GitHub releases with artifacts and generates release notes based on git commit history between releases.

## Table of Contents
- [Overview](#overview)
- [Inputs](#inputs)
- [Usage](#usage)
- [Workflow Logic](#workflow-logic)
- [Script Details](#script-details)
  - [create-update-release.sh](#create-update-releasesh)
- [Release Notes Generation](#release-notes-generation)
- [Dependencies](#dependencies)
- [Edge Cases](#edge-cases)
- [Troubleshooting](#troubleshooting)

## Overview

This composite action provides automated GitHub release management with the following capabilities:

1. **Artifact Management** - Automatically zips and uploads release artifacts
2. **Release Creation/Updates** - Creates new releases or updates existing ones
3. **Cross-Platform Support** - Handles both Windows and Unix-based runners
4. **Automatic Release Notes** - Generates release notes from git commit history
5. **Large Content Handling** - Manages release notes that exceed GitHub's size limits

The action intelligently detects existing releases and updates them with new artifacts while preserving release history and generating comprehensive release notes.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `name` | The name of the release (commonly version number). Creates a new tag with this name. | **Yes** | - |
| `target` | The branch or tag to create the release from | **Yes** | - |
| `artifacts` | A set of paths to the artifacts to upload to the release | **Yes** | - |
| `artifacts-directory` | Parent directory where artifact paths will be resolved from | No | `./` |
| `artifact-name` | Name of the generated zip file containing all release artifacts | No | `artifact` |
| `prerelease` | Flag to mark the release as a prerelease | No | `true` |

## Usage

### Basic Usage

```yaml
- name: Create or Update Release
  uses: fbitn/github-composite-actions/release/create-update-release@main
  with:
    name: 'v1.0.0'
    target: 'main'
    artifacts: 'dist/ build/ docs/'
    prerelease: 'false'
```

## Workflow Logic

The action follows a structured approach:

1. **Artifact Compression**
   - **Windows**: Uses PowerShell `Compress-Archive` cmdlet
   - **Non-Windows**: Uses standard `zip` command
   - Creates a single zip file containing all specified artifacts

2. **Release Management**
   - Attempts to create a new GitHub release
   - If release already exists (HTTP 422 error), updates existing release
   - Uploads artifacts with `--clobber` flag to replace existing files

3. **Release Notes Generation**
   - Determines commit range between latest release and target branch
   - Generates formatted commit log
   - Handles large release notes by attaching as files

## Script Details

### create-update-release.sh

**Purpose**: Core script that orchestrates GitHub release creation, artifact upload, and release notes generation.

#### Tools Used
- **GitHub CLI (`gh`)**: Release management, artifact uploads, API interactions
- **git**: Commit history analysis and branch operations
- **bash**: Shell scripting with error handling and conditional logic

#### Parameters
1. `RELEASE_NAME` - Name/version of the release
2. `BRANCH_NAME` - Target branch for the release
3. `ARTIFACTS_ZIP` - Path to the zipped artifacts file
4. `PRE_RELEASE` - Boolean flag for prerelease status

#### Logic Flow

1. **Parameter Validation**
   ```bash
   if [ -z "${RELEASE_NAME}" ]; then
       echo "ERROR :: Release name is required."
       exit 1
   fi
   ```
   - Validates all required parameters are provided
   - Exits with error code 1 if any parameter is missing
   - Provides descriptive error messages for debugging

2. **Release Creation Strategy**
   ```bash
   CREATE_RELEASE_STATUS=$(gh release create $RELEASE_NAME --title $RELEASE_NAME --target $BRANCH_NAME --prerelease=$PRE_RELEASE 2>&1)
   ```
   - Attempts to create new GitHub release
   - Captures both stdout and stderr output
   - Uses release name as both tag and title
   - Applies prerelease flag as specified

3. **Existing Release Handling**
   ```bash
   if [[ $CREATE_RELEASE_STATUS == *"422"* ]]; then
       echo "Release $RELEASE_NAME already exists. Updating with new artifacts and release notes."
   ```
   - Detects HTTP 422 status indicating existing release
   - Gracefully handles duplicate release scenarios
   - Continues with update workflow instead of failing

4. **Artifact Upload Process**
   ```bash
   gh release upload $RELEASE_NAME $ARTIFACTS_ZIP --clobber
   ```
   - Uploads zipped artifacts to the release
   - Uses `--clobber` flag to overwrite existing files
   - Maintains artifact history while allowing updates

5. **Release Notes Generation Algorithm**
   ```bash
   LATEST_RELEASE_TAG=$(gh release list --exclude-drafts --exclude-pre-releases --json isLatest,tagName --jq '.[]| select(.isLatest)|.tagName')
   ```
   - Queries for the latest stable release (excluding drafts and prereleases)
   - Uses jq to parse JSON response and extract tag name
   - Determines commit range for release notes

6. **Commit Range Determination**
   ```bash
   if [ -z "${LATEST_RELEASE_TAG}" ]; then
       LOG=$(git log $BRANCH_NAME --pretty=format:"%s by %aN in %h" --no-merges)
   else
       LOG=$(git log $LATEST_RELEASE_TAG..$BRANCH_NAME --pretty=format:"%s by %aN in %h" --no-merges)
   fi
   ```
   - **First Release**: Uses entire branch history
   - **Subsequent Releases**: Uses commits between latest release and target branch
   - Excludes merge commits with `--no-merges` flag
   - Formats commits with subject, author, and short hash

7. **Large Content Management**
   ```bash
   LOG_CHARACTERS_COUNT=$(wc -c< release-notes.log)
   if [[ "${LOG_CHARACTERS_COUNT}" -gt 125000 ]]; then
       gh release edit $RELEASE_NAME --notes 'Release notes added as attachment - release-notes.log'
       gh release upload $RELEASE_NAME release-notes.log
   else
       gh release edit $RELEASE_NAME --notes-file release-notes.log
   fi
   ```
   - Measures release notes character count
   - **Small Content** (≤125,000 chars): Embeds directly in release description
   - **Large Content** (>125,000 chars): Attaches as separate file
   - Prevents GitHub API errors from oversized release descriptions

#### Advanced Features

**Cross-Platform Compatibility**
- Windows PowerShell integration
- Unix/Linux zip command support
- Path resolution handling across operating systems

**Error Recovery**
- Graceful handling of existing releases
- Comprehensive parameter validation
- Descriptive error messages for troubleshooting

**Performance Optimization**
- Efficient commit log generation
- Single API calls where possible
- Minimal file I/O operations

## Release Notes Generation

### Commit Log Format
```
Add new authentication module by John Doe in abc123f
Fix critical security vulnerability by Jane Smith in def456a
Update documentation for API endpoints by Bob Johnson in 789xyz1
```

### Format Components
- **Subject**: Full commit message subject line
- **Author**: Git author name (`%aN`)
- **Hash**: Abbreviated commit hash (`%h`)
- **Separator**: " by " and " in " for readability

### Exclusions
- **Merge Commits**: Automatically filtered out with `--no-merges`
- **Draft Releases**: Not considered for commit range calculation
- **Prerelease Tags**: Excluded when determining latest stable release

## Dependencies

### Required Tools
- **GitHub CLI**: Version 2.0+ with authentication
- **git**: Version control system with commit history access
- **PowerShell**: Windows environments only (for artifact compression)
- **zip**: Unix/Linux environments (standard utility)
- **bash**: Shell interpreter with process substitution support

### Required Permissions
- **Repository**: Read access for git operations and artifact paths
- **Releases**: Write access for creating and updating releases  
- **Contents**: Read access for artifact file access
- **Actions**: Standard GitHub Actions token permissions

### Environment Requirements
- **GitHub Token**: Automatically provided via `${{ github.token }}`
- **File System**: Write access in runner workspace
- **Network**: Connectivity to GitHub API and release endpoints

## Troubleshooting

### Common Issues

1. **"422 Unprocessable Entity" Errors**
   ```
   Solution: Release already exists. Action will automatically update it.
   This is expected behavior and not an error.
   ```

2. **"Artifact not found" Errors**
   ```
   Cause: Specified artifact paths don't exist
   Solution: Verify artifact paths and ensure build steps complete successfully
   Debug: List files before running action with `ls -la` or `dir`
   ```

3. **"Permission denied" Errors**
   ```
   Cause: Insufficient GitHub token permissions
   Solution: Ensure GITHUB_TOKEN has releases:write permission
   Check: Repository settings > Actions > General > Workflow permissions
   ```

4. **Empty Release Notes**
   ```
   Cause: No commits between releases or all commits are merges
   Solution: Check commit history with `git log --no-merges`
   ```

5. **Zip Creation Failures**
   ```
   Windows: Ensure PowerShell execution policy allows Compress-Archive
   Linux: Verify zip utility is installed and paths are accessible
   ```


### Local Testing

```bash
# Set up environment
export GH_TOKEN="your-github-token"

# Create test artifacts
mkdir -p test-artifacts
echo "test content" > test-artifacts/test.txt

# Zip artifacts (Linux/Mac)
zip -r artifact.zip test-artifacts/

# Run script directly
bash create-update-release.sh "test-release" "main" "artifact.zip" "true"
```

### Validation Checklist

- [ ] All required inputs provided
- [ ] Artifact paths exist and are accessible  
- [ ] GitHub token has sufficient permissions
- [ ] Target branch exists in repository

### Release Notes Size Limits

```bash
# Check release notes size before upload
wc -c release-notes.log

# If > 125,000 characters:
# - Release notes will be attached as file
# - Release description will contain reference to attachment
# - This is automatic behavior, no action required
```