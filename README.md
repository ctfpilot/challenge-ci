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
| [CTF Pilot's Challenge Toolkit](https://github.com/ctfpilot/challenge-toolkit) | v2.0+   |
| [CTF Pilot's Challenge Schema](https://github.com/ctfpilot/challenge-schema)   | v1.0+   |
| [CTF Pilot's Page Schema](https://github.com/ctfpilot/page-schema)             | v1.0+   |
| [kube-ctf](https://github.com/ctfpilot/kube-ctf)                               | v1.0+   |

## Workflows

The workflows use the CTF Pilot's Challenge Toolkit for processing challenges and pages.
This toolkit is available at [ctfpilot/challenge-toolkit](https://github.com/ctfpilot/challenge-toolkit).

The workflows are by default locked to a specific version (currently `v2.0.0`) of the Challenge Toolkit, but it can be overridden with the `toolkit-cli`, `toolkit-install-package`, and `toolkit-version` inputs, allowing you to use a custom version or installation of the toolkit if desired.

> [!IMPORTANT]
> Since v2.0 of the Challenge CI, the Challenge toolkit is no longer used from a local installation, but is instead installed as a package using uv.
> If you are upgrading and use a custom Challenge Toolkit, you will need to make your custom Challenge Toolkit installation available to download using uv and update your workflow calls to specify the `toolkit-install-package` and `toolkit-version` inputs. The `toolkit-version` may be set to an empty string, to allow for installing from git.

### Create Challenge

**Workflow**: [`create-chall.yml`](.github/workflows/create-chall.yml)  

Creates a new challenge with all necessary scaffolding and metadata. This workflow:

- Initializes challenge directory structure
- Generates challenge metadata file
- Creates a new branch and pull request
- Creates or links to an associated GitHub issue
- Creates GitHub labels for the challenge
- Creates GitHub milestone if it doesn't exist

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
    # Please change `@main` to a specific version tag (e.g., `@v1.0.2`)
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
```

See the full example in [`examples/create-chall.yml`](examples/create-chall.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.2` to pin to version 1.0.2 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

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
- `toolkit-version`: Version of Challenge Toolkit to use (default: `v2.0.0`). When not an empty string, binds the toolkit install package to a specific version.
- `toolkit-install-package`: Package name to install Challenge Toolkit from (default: `challenge-toolkit`)
- `toolkit-cli`: Challenge Toolkit CLI command (default: `challenge-toolkit`)
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
    # Please change `@main` to a specific version tag (e.g., `@v1.0.2`)
    uses: ctfpilot/challenge-ci/.github/workflows/configure-challs.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
    with:
      runs-on: "ubuntu-latest"
```

See the full example in [`examples/configure-challs.yml`](examples/configure-challs.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.2` to pin to version 1.0.2 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-version`: Version of Challenge Toolkit to use (default: `v2.0.0`). When not an empty string, binds the toolkit install package to a specific version.
- `toolkit-install-package`: Package name to install Challenge Toolkit from (default: `challenge-toolkit`)
- `toolkit-cli`: Challenge Toolkit CLI command (default: `challenge-toolkit`)
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
    # Please change `@main` to a specific version tag (e.g., `@v1.0.2`)
    uses: ctfpilot/challenge-ci/.github/workflows/configure-pages.yml@main
    permissions:
      contents: write
    with:
      runs-on: "ubuntu-latest"
```

See the full example in [`examples/configure-pages.yml`](examples/configure-pages.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.2` to pin to version 1.0.2 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-version`: Version of Challenge Toolkit to use (default: `v2.0.0`). When not an empty string, binds the toolkit install package to a specific version.
- `toolkit-install-package`: Package name to install Challenge Toolkit from (default: `challenge-toolkit`)
- `toolkit-cli`: Challenge Toolkit CLI command (default: `challenge-toolkit`)
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
    # Please change `@main` to a specific version tag (e.g., `@v1.0.2`)
    uses: ctfpilot/challenge-ci/.github/workflows/render-templates.yml@main
    permissions:
      contents: write
    with:
      runs-on: "ubuntu-latest"
```

See the full example in [`examples/render-templates.yml`](examples/render-templates.yml).  
Place the workflow in your challenge repository's `.github/workflows/` directory.

> [!IMPORTANT]
> To prevent supply chain attacks, please always pin the version of the workflow you are using.  
> Our examples use `@main` for simplicity, but must be changed to a specific version tag in production.  
> For example, use `@v1.0.2` to pin to version 1.0.2 of the workflow. [See available versions](https://github.com/ctfpilot/challenge-ci/releases).

**Optional inputs**:

- `runs-on`: GitHub Actions runner (default: `ubuntu-latest`)
- `toolkit-version`: Version of Challenge Toolkit to use (default: `v2.0.0`). When not an empty string, binds the toolkit install package to a specific version.
- `toolkit-install-package`: Package name to install Challenge Toolkit from (default: `challenge-toolkit`)
- `toolkit-cli`: Challenge Toolkit CLI command (default: `challenge-toolkit`)
- `fetch-submodules`: Specify if submodules should be fetched (default: `true`)

**Permissions required**:

- `contents`: write (to commit generated files)

## Setup Instructions

### Prerequisites

- A GitHub repository with your CTF challenges, following the structure outlined in the [Challenge Repository Structure](#challenge-repository-structure) section below
- Appropriate GitHub secrets and repository configuration

### Integration Steps

1. **Create workflow files** in your challenge repository at `.github/workflows/`:
   - Use the examples provided in the [`examples/`](examples/) directory
   - Customize the inputs to match your setup

2. **Set up Docker authentication** (for `configure-challs.yml`):
   - Ensure your GitHub Actions workflow has write access to GHCR
   - Configure appropriate permissions in your repository

3. **Ensure templates are ready**:
   - Copy the toolkit templates into `template/` directory of your repository. Templates can be found in the [Challenge Toolkit repository](https://github.com/ctfpilot/challenge-toolkit/tree/main/template).

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
- **`toolkit-version`**: Version of Challenge Toolkit to use (default: `v2.0.0`). When not an empty string, binds the toolkit install package to a specific version.
- **`toolkit-install-package`**: Package name to install Challenge Toolkit from (default: `challenge-toolkit`)
- **`toolkit-cli`**: Challenge Toolkit CLI command (default: `challenge-toolkit`)
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

This repository is licensed under the **EUPL-1.2 License**.  
You can find the full license in the **[LICENSE](LICENSE)** file.

We encourage all modifications and contributions to be shared back with the community, for example through pull requests to this repository.  
We also encourage all derivative works to be publicly available under the **EUPL-1.2 License**.  
At all times must the license terms be followed.

For information regarding how to contribute, see the [contributing](#contributing) section above.

CTF Pilot is owned and maintained by **[The0Mikkel](https://github.com/The0mikkel)**.  
Required Notice: Copyright Mikkel Albrechtsen (<https://themikkel.dk>)

## Code of Conduct

We expect all contributors to adhere to our [Code of Conduct](/CODE_OF_CONDUCT.md) to ensure a welcoming and inclusive environment for all.
