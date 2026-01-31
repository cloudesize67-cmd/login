# Azure GitHub Actions Manager Interface

This interface provides a powerful way to manage GitHub Actions workflows through Azure integration. It is implemented as a reusable workflow that enables you to:

- **List workflows** across different repositories
- **Trigger workflows** remotely with custom inputs
- **Monitor workflow runs** and their status
- **Manage multiple repositories** from a single interface
- **Integrate with Azure** for enhanced security and authentication

## Features

- 🔐 Secure Azure authentication (OIDC, Service Principal, or Managed Identity)
- 🚀 Remote workflow triggering with custom inputs
- 📊 Workflow status monitoring and management
- 🔄 Multi-repository support
- 🛡️ GitHub CLI integration for reliable API interactions
- 📝 Comprehensive logging and error handling
- ♻️ Reusable workflow pattern for easy integration

## Quick Start

### Prerequisites

1. **Azure Setup** (if using Azure authentication):
   - Azure Service Principal or Managed Identity
   - Appropriate Azure permissions
   - GitHub secrets configured for Azure credentials

2. **GitHub Setup**:
   - GitHub token with `actions:read` and `actions:write` permissions
   - For triggering workflows: `workflow:write` permission

### Basic Usage

The Azure GitHub Actions Manager is implemented as a reusable workflow. Here's how to use it:

#### Example 1: List Workflows in Current Repository

```yaml
name: List Workflows
on: [workflow_dispatch]

permissions:
  id-token: write
  contents: read
  actions: read

jobs:
  list-workflows:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: list-workflows
    secrets:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
```

#### Example 2: Trigger a Workflow in Another Repository

```yaml
name: Trigger Remote Workflow
on: [workflow_dispatch]

permissions:
  id-token: write
  contents: read
  actions: write

jobs:
  trigger-workflow:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      operation: trigger-workflow
      target-repository: owner/repository
      workflow-file: deploy.yml
      workflow-ref: main
      workflow-inputs: '{"environment": "production"}'
    secrets:
      AZURE_CREDS: ${{ secrets.AZURE_CREDENTIALS }}
      GITHUB_PAT: ${{ secrets.GH_PAT }}
```

#### Example 3: Monitor Workflow Runs

```yaml
name: Monitor Workflows
on: [workflow_dispatch]

permissions:
  contents: read
  actions: read

jobs:
  monitor:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: get-workflow-runs
      workflow-file: ci.yml
    secrets:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
```

#### Example 4: Manage Multiple Repositories

```yaml
name: Manage Repositories
on: [workflow_dispatch]

permissions:
  contents: read
  actions: read

jobs:
  manage:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: manage-repositories
    secrets:
      GITHUB_PAT: ${{ secrets.GH_PAT }}
```

## Input Parameters

### Azure Authentication Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `client-id` | No | - | Azure Service Principal client ID |
| `tenant-id` | No | - | Azure tenant ID |
| `subscription-id` | No | - | Azure subscription ID |
| `creds` | No | - | Azure credentials in JSON format |
| `enable-AzPSSession` | No | `false` | Enable Azure PowerShell login |
| `environment` | No | `azurecloud` | Azure environment |

### GitHub Actions Management Inputs (Workflow Inputs)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `target-repository` | No | Current repo | Target repository (owner/repo) |
| `workflow-file` | No | - | Workflow file name (e.g., ci.yml) |
| `workflow-ref` | No | `main` | Git reference (branch, tag, or SHA) |
| `workflow-inputs` | No | `{}` | Workflow inputs as JSON string |
| `operation` | No | `list-workflows` | Operation to perform |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AZURE_CREDS` | No | Azure credentials in JSON format (alternative to individual parameters) |
| `GITHUB_PAT` | No | GitHub Personal Access Token with workflow permissions. If not provided, uses default GITHUB_TOKEN |

### Operations

| Operation | Description | Required Inputs |
|-----------|-------------|-----------------|
| `list-workflows` | List all workflows in a repository | `target-repository` |
| `trigger-workflow` | Trigger a specific workflow | `workflow-file`, `target-repository` |
| `get-workflow-runs` | Get recent workflow runs | Optional: `workflow-file` |
| `manage-repositories` | List accessible repositories | None |

## Output Parameters

| Output | Description |
|--------|-------------|
| `workflow-status` | Status of the operation (success, triggered, etc.) |

**Note**: The workflow-run-id is not currently captured as GitHub CLI doesn't return it immediately for async workflow runs.

## Advanced Usage

### Using with Azure Integration

```yaml
name: Azure Integrated Workflow Management
on: [workflow_dispatch]

permissions:
  id-token: write
  contents: read
  actions: write

jobs:
  # Note: Azure authentication is done within the reusable workflow
  deploy-with-azure:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      # Azure OIDC authentication
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      
      # Trigger deployment workflow
      operation: trigger-workflow
      target-repository: my-org/my-app
      workflow-file: azure-deploy.yml
      workflow-ref: main
      workflow-inputs: '{"azure_resource_group": "production-rg"}'
    secrets:
      AZURE_CREDS: ${{ secrets.AZURE_CREDENTIALS }}
      GITHUB_PAT: ${{ secrets.GH_PAT }}
  
  # Separate job to use Azure CLI after authentication
  use-azure-cli:
    runs-on: ubuntu-latest
    needs: deploy-with-azure
    steps:
      - uses: actions/checkout@v4
      
      - name: Azure Login
        uses: ./
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      
      - name: Use Azure CLI
        run: |
          az account show
          az group list
```

### Multi-Repository Workflow Coordination

```yaml
name: Coordinate Multiple Repositories
on: [workflow_dispatch]

permissions:
  contents: read
  actions: write

jobs:
  trigger-frontend:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: trigger-workflow
      target-repository: my-org/frontend
      workflow-file: deploy.yml
      workflow-ref: main
    secrets:
      GITHUB_PAT: ${{ secrets.GH_PAT }}
  
  trigger-backend:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: trigger-workflow
      target-repository: my-org/backend
      workflow-file: deploy.yml
      workflow-ref: main
    secrets:
      GITHUB_PAT: ${{ secrets.GH_PAT }}
  
  trigger-infrastructure:
    uses: ./.github/workflows/azure-actions-manager-reusable.yml
    with:
      operation: trigger-workflow
      target-repository: my-org/infrastructure
      workflow-file: terraform-apply.yml
      workflow-ref: main
    secrets:
      GITHUB_PAT: ${{ secrets.GH_PAT }}
```

## Security Considerations

1. **GitHub Token Permissions**: Ensure your GitHub token has minimal required permissions:
   - `actions:read` for listing workflows and runs
   - `actions:write` for triggering workflows
   - Store tokens as GitHub secrets

2. **Azure Authentication**: Use OIDC-based authentication when possible for enhanced security:
   - No long-lived credentials
   - Automatic token rotation
   - Reduced secret management overhead

3. **Repository Access**: Be cautious when using tokens with access to multiple repositories:
   - Use fine-grained personal access tokens when possible
   - Limit repository access scope
   - Regularly rotate tokens

## Troubleshooting

### GitHub CLI Not Found

If you encounter "GitHub CLI is not installed" error:
- Ensure you're using a GitHub-hosted runner (ubuntu-latest, windows-latest, macos-latest)
- GitHub CLI is pre-installed on all GitHub-hosted runners

### Authentication Errors

If you encounter authentication errors:
1. Verify your GitHub token has the required permissions
2. Check that Azure credentials are correctly configured
3. Ensure the target repository exists and is accessible

### Workflow Not Triggering

If the workflow doesn't trigger:
1. Verify the workflow file exists in the target repository
2. Check that the workflow has a `workflow_dispatch` trigger
3. Ensure your token has `actions:write` permission
4. Verify the workflow-ref (branch/tag) exists

## Examples Repository

For more comprehensive examples and use cases, check out the demo workflow:
- [.github/workflows/azure-actions-manager.yml](.github/workflows/azure-actions-manager.yml)

## Contributing

Contributions are welcome! Please ensure:
- All changes maintain backward compatibility
- Documentation is updated
- Security best practices are followed

## License

This project inherits the license from the Azure Login Action (MIT License).

## Support

For issues and questions:
- Open an issue in this repository
- Refer to the [Azure Login Action documentation](README.md)
- Check [GitHub Actions documentation](https://docs.github.com/en/actions)

## Related Actions

- [Azure Login](README.md) - The base Azure login action
- [Azure CLI Action](https://github.com/Azure/cli) - Run Azure CLI scripts
- [Azure PowerShell Action](https://github.com/Azure/powershell) - Run Azure PowerShell scripts
