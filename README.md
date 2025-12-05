# CTF Pilot's Challenge CI

A centralized GitHub Actions CI system for managing CTF challenges and pages within the CTF Pilot infrastructure.  
This repository provides reusable workflows that automate challenge creation, configuration, deployment, and template rendering.

## Overview

> [!IMPORTANT]
> This repository uses the [CTF Pilot's Challenge Toolkit](https://github.com/ctfpilot/challenge-toolkit) for challenge management and template rendering.  
> Please see its documentation for detailed information about challenge structure and schema.

This CI system serves as the backbone for managing challenges in the CTF Pilot ecosystem. It provides automated workflows that:

- **Create new challenges** with standardized structure and metadata
- **Configure challenges** by building Docker images and rendering Kubernetes deployment files
- **Configure pages** for CTFd integration
- **Render templates** across all challenges and pages

The workflows are designed to be called from your challenge repository, enabling seamless integration with your CTF infrastructure.

> [!NOTE]
> The `release.yml` and `cla-assistant.yml` workflows in the `.github/workflows` directory are the CI for this repository itself and are not part of the Challenge CI system provided here.

## Supported CTF Pilot Components

| Component                                                                      | Version |
| ------------------------------------------------------------------------------ | ------- |
| [CTF Pilot's Challenge Toolkit](https://github.com/ctfpilot/challenge-toolkit) | v1.0+   |
| [CTF Pilot's Challenge Schema](https://github.com/ctfpilot/challenge-schema)   | v1.0+   |
| [CTF Pilot's Page Schema](https://github.com/ctfpilot/page-schema)             | v1.0+   |
| [kube-ctf](https://github.com/ctfpilot/kube-ctf)                               | v1.0+   |

## Workflows

### Create Challenge

**Workflow**: [`create-chall.yml`](.github/workflows/create-chall.yml)  

Creates a new challenge with all necessary scaffolding and metadata. This workflow:

- Initializes challenge directory structure
- Generates challenge metadata file
- Creates a new branch and pull request
- Creates or links to an associated GitHub issue
- Creates GitHub labels for the challenge

**When to use**: When setting up a new challenge.

**Example usage**:

```yaml
name: Create challenge
on:
  workflow_dispatch:
    inputs:
      name:
        description: "Name of the challenge"
        required: true
        type: string
      author:
        description: "Author of the challenge"
        required: true
        type: string
      category:
        description: "Challenge category"
        required: true
        type: choice
        options: [web, forensics, rev, crypto, pwn, misc]
      difficulty:
        required: true
        type: choice
        options: [beginner, easy, medium, hard, very-hard, insane]
      type:
        description: "Challenge type"
        required: true
        type: choice
        options: [static, shared, instanced]
      flag:
        # replace `FLAG` with your flag prefix
        description: "Challenge flag (format: FLAG{...}, dynamic, or null)"
        required: true
        type: string
      points:
        description: "Maximum points"
        required: false
        type: number
        default: 1000

jobs:
  create:
    # Please change `@main` to a specific version tag (e.g., `@v1.0.0`)
    uses: ctfpilot/challenge-ci/.github/workflows/create-chall.yml@main
    permissions:
      contents: write
      pull-requests: write
      issues: write
    with:
      name: ${{ inputs.name }}
      author: ${{ inputs.author }}
      category: ${{ inputs.category }}
      difficulty: ${{ inputs.difficulty }}
      type: ${{ inputs.type }}
      flag: ${{ inputs.flag }}
      points: ${{ inputs.points }}
      toolkit-path: "./challenge-toolkit/"
```

See the full example in [`examples/create-chall.yml`](examples/create-chall.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.0` to pin to version 1.0.0 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Required inputs**:

- `name`: Challenge name
- `author`: Challenge author (alias)
- `category`: Challenge category
- `difficulty`: Challenge difficulty
- `type`: Challenge type (static, shared, instanced)
- `flag`: Challenge flag

**Optional inputs**:

- `issue`: Link to existing GitHub issue (default: `0`, creates new)
- `instanced_type`: Type for instanced challenges (`web`, `tcp`, `none`; default: `none`)
- `min-points`: Minimum points (default: `10`)
- `points`: Maximum points (default: `1000`)
- `pr-base`: Base branch for PR (default: `develop`)
- `milestone`: GitHub milestone (default: `Challenges`)
- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-path`: Path to Challenge Toolkit (default: `./challenge-toolkit/`)
- `fetch-submodules`: Specify if submodules should be fetched (default: `true`)

**Permissions required**:

- `contents`: write (to create branches and PRs)
- `pull-requests`: write (to create PRs)
- `issues`: write (to create/link issues)

### Configure Challenges

**Workflow**: [`configure-challs.yml`](.github/workflows/configure-challs.yml)

Automatically processes challenge changes by:

1. Detecting which challenges have been modified since last commit on the current branch
2. Building Docker images for changed challenges
3. Pushing images to GitHub Container Registry (GHCR)
4. Rendering Kubernetes deployment templates
5. Committing generated files back to the repository

This workflow uses path-based filtering to only process challenges that have actually changed since the last commit, optimizing build times and resource usage.

**When to use**: Triggered automatically on push to challenges directory, or manually via workflow dispatch.  
*You may limit the pipeline to only run on specific branches if desired.*

**Example usage**:

```yaml
name: Configure challenges
on:
  push:
    paths:
      - "challenges/**"
  workflow_dispatch:

jobs:
  configure:
    # Please change `@main` to a specific version tag (e.g., `@v1.0.0`)
    uses: ctfpilot/challenge-ci/.github/workflows/configure-challs.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
    with:
      runs-on: "ubuntu-latest"
      toolkit-path: "./challenge-toolkit/"
```

See the full example in [`examples/configure-challs.yml`](examples/configure-challs.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.0` to pin to version 1.0.0 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-path`: Path to Challenge Toolkit (default: `./challenge-toolkit/`)
- `fetch-submodules`: Specify if submodules should be fetched (default: `true`)

**Permissions required**:

- `contents`: write (to commit generated files)
- `packages`: write (to push Docker images)
- `id-token`: write (for OIDC authentication)

### Configure Pages

**Workflow**: [`configure-pages.yml`](.github/workflows/configure-pages.yml)

Processes page changes by:

1. Detecting which pages have been modified
2. Rendering page templates using the Challenge Toolkit
3. Committing generated files back to the repository

This workflow uses path-based filtering to only process pages that have actually changed since the last commit, optimizing build times and resource usage.

**When to use**: Triggered automatically on push to pages directory, or manually via workflow dispatch.  
*You may limit the pipeline to only run on specific branches if desired.*

**Example usage**:

```yaml
name: Configure pages
on:
  push:
    paths:
      - "pages/**"
  workflow_dispatch:

jobs:
  configure:
    # Please change `@main` to a specific version tag (e.g., `@v1.0.0`)
    uses: ctfpilot/challenge-ci/.github/workflows/configure-pages.yml@main
    permissions:
      contents: write
    with:
      runs-on: "ubuntu-latest"
      toolkit-path: "./challenge-toolkit/"
```

See the full example in [`examples/configure-pages.yml`](examples/configure-pages.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.0` to pin to version 1.0.0 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-path`: Path to Challenge Toolkit (default: `./challenge-toolkit/`)
- `fetch-submodules`: Specify if submodules should be fetched (default: `true`)

**Permissions required**:

- `contents`: write (to commit generated files)

### Render All Templates

**Workflow**: [`render-templates.yml`](.github/workflows/render-templates.yml)

Renders templates for all challenges and pages in the repository. Useful for:

- Initial setup or migration
- Bulk updates to all challenges
- Regenerating all deployment templates

*This does not update versions or build Docker images; use [`configure-challs.yml`](#configure-challenges) for that.*

**When to use**: Manual workflow dispatch, typically as an administrative task.

**Example usage**:

```yaml
name: Render all templates
on:
  workflow_dispatch:

jobs:
  render:
    # Please change `@main` to a specific version tag (e.g., `@v1.0.0`)
    uses: ctfpilot/challenge-ci/.github/workflows/render-templates.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
    with:
      runs-on: "ubuntu-latest"
      toolkit-path: "./challenge-toolkit/"
```

See the full example in [`examples/render-templates.yml`](examples/render-templates.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.0` to pin to version 1.0.0 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-path`: Path to Challenge Toolkit (default: `./challenge-toolkit/`)
- `fetch-submodules`: Specify if submodules should be fetched (default: `true`)

**Permissions required**:

- `contents`: write (to commit generated files)

## Setup Instructions

### Prerequisites

- A GitHub repository with your CTF challenges
- [CTF Pilot's Challenge Toolkit](https://github.com/ctfpilot/challenge-toolkit) installed as a submodule or included in your repository
- Appropriate GitHub secrets and repository configuration

### Integration Steps

1. **Create workflow files** in your challenge repository at `.github/workflows/`:
   - Use the examples provided in the [`examples/`](examples/) directory
   - Customize the inputs to match your setup

2. **Set up Docker authentication** (for `configure-challs.yml`):
   - Ensure your GitHub Actions workflow has write access to GHCR
   - Configure appropriate permissions in your repository

3. **Ensure Challenge Toolkit is available**:
   - Add as a git submodule: `git submodule add https://github.com/ctfpilot/challenge-toolkit`
   - Or copy the toolkit directory into your repository
   - Update `toolkit-path` in workflow calls if needed

4. **Commit and test**:
   - Test workflows manually via GitHub Actions UI
   - Verify Docker images are built correctly
   - Check that Kubernetes manifests are generated

### Challenge Repository Structure

For these workflows to function properly, your challenge repository should follow this structure:

```text
.
├── challenges/
│   └── <category>/
│       └── <challenge-slug>/
├── pages/
│   └── <page-slug>/
├── template/
├── challenge-toolkit/
└── <other files>
```

For detailed information about challenge structure and schema, see the [Challenge Toolkit documentation](https://github.com/ctfpilot/challenge-toolkit).

## Customization

All workflows support customization through inputs:

- **`runs-on`**: Specify different GitHub Actions runners (e.g., `ubuntu-latest`, self-hosted runner)
- **`toolkit-path`**: Adjust if your Challenge Toolkit is in a non-standard location
- **`fetch-submodules`**: Specify if submodules should be fetched (default: `true`)

For more advanced customization, fork this repository and modify the workflows as needed, then reference your fork in your challenge repository workflows.

If you believe that your changes would benefit the community, consider submitting a pull request back to this repository!  
See the [contributing](#contributing) section below for more information.

## Troubleshooting

### Docker Build Failures

If `configure-challs.yml` fails during Docker builds:

- Verify your Dockerfile is valid, that it is in the challenge directory, and that it is correctly configured in the `challenge.yaml`
- Check for missing dependencies or incorrect base images
- Review GitHub Actions logs for specific error messages

### Template Rendering Issues

If template rendering fails:

- Verify your `challenge.yaml` or `page.yaml` matches the schema
- Check that deployment templates exist in the `template/` directory
- Review the [Challenge Toolkit documentation](https://github.com/ctfpilot/challenge-toolkit) for template requirements
- Review GitHub Actions logs for specific error messages

### Git Push Conflicts

The workflows include retry logic to handle concurrent pushes, but if conflicts persist:

- Ensure only one workflow is pushing at a time
- Check for manual changes conflicting with CI commits
- Manually bump the version in the `version` file if necessary, as it will trigger a new build

## Related Documentation

- [CTF Pilot's Challenge Toolkit](https://github.com/ctfpilot/challenge-toolkit)
- [CTF Pilot's Challenge Schema](https://github.com/ctfpilot/challenge-schema)
- [CTF Pilot's Page Schema](https://github.com/ctfpilot/page-schema)
- [kube-ctf](https://github.com/ctfpilot/kube-ctf)

## Contributing

We welcome contributions of all kinds, from **code** and **documentation** to **bug reports** and **feedback**!

Please check the [Contribution Guidelines (`CONTRIBUTING.md`)](/CONTRIBUTING.md) for detailed guidelines on how to contribute.

To maintain the ability to distribute contributions across all our licensing models, **all code contributions require signing a Contributor License Agreement (CLA)**.
You can review **[the CLA here](https://github.com/ctfpilot/cla)**. CLA signing happens automatically when you create your first pull request.  
To administrate the CLA signing process, we are using **[CLA assistant lite](https://github.com/marketplace/actions/cla-assistant-lite)**.

*A copy of the CLA document is also included in this repository as [`CLA.md`](CLA.md).*  
*Signatures are stored in the [`cla` repository](https://github.com/ctfpilot/cla).*

## License

This schema and repository is licensed under the **EUPL-1.2 License**.  
You can find the full license in the **[LICENSE](LICENSE)** file.

We encourage all modifications and contributions to be shared back with the community, for example through pull requests to this repository.  
We also encourage all derivative works to be publicly available under the **EUPL-1.2 License**.  
At all times must the license terms be followed.

For information regarding how to contribute, see the [contributing](#contributing) section above.

CTF Pilot is owned and maintained by **[The0Mikkel](https://github.com/The0mikkel)**.  
Required Notice: Copyright Mikkel Albrechtsen (<https://themikkel.dk>)

## Code of Conduct

We expect all contributors to adhere to our [Code of Conduct](/CODE_OF_CONDUCT.md) to ensure a welcoming and inclusive environment for all.
