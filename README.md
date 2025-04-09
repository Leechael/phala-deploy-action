# Phala Cloud CVM Deployment Action

This GitHub Action helps you easily deploy Container Virtual Machines (CVMs) to Phala Cloud.

## Features

- Automatic setup of Bun runtime environment
- Installation of Phala CLI tools
- Authentication to Phala Cloud using your API key
- Creation of CVMs with specified configurations
- Support for updating existing CVMs

## Usage

### Basic Usage

```yaml
name: Deploy to Phala Cloud

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Phala Cloud
        uses: Leechael/phala-deploy-action@v2
        with:
          phala-api-key: ${{ secrets.PHALA_CLOUD_API_KEY }}
```

### Complete Configuration Options

```yaml
- name: Deploy to Phala Cloud
  uses: Leechael/phala-deploy-action@v2
  with:
    # Required parameters
    phala-api-key: ${{ secrets.PHALA_CLOUD_API_KEY }}
    
    # Optional parameters (with defaults)
    cvm-name: 'my-app'                # Default: repository name (sanitized) or 'my-dstack-app'
    compose-file: './my-compose.yml'  # Default: './docker-compose.yml'
    vcpu: '4'                         # Default: '2'
    memory: '4096'                    # Default: '2048'
    disk-size: '10'                   # Default: '40'
    envs: |                           # Environment variables in YAML format
      DB_HOST: localhost
      DB_PORT: 5432
    app-id: ''                        # App ID of existing CVM to update
    node-id: ''                       # Node ID (Teepod ID)
    base-image: ''                    # Base image to use for the CVM
```

## Input Parameters

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `phala-api-key` | Phala Cloud API Key for authentication | Yes | - |
| `cvm-name` | Name for the CVM | No | Repository name (sanitized) or 'my-dstack-app' |
| `compose-file` | Path to docker-compose.yml file | No | './docker-compose.yml' |
| `vcpu` | Number of virtual CPUs | No | '2' |
| `memory` | Memory size in MB | No | '2048' |
| `disk-size` | Disk size in GB | No | '40' |
| `envs` | Environment variables in YAML format | No | '' |
| `app-id` | App ID of existing CVM to update | No | '' |
| `node-id` | Node ID (Teepod ID) | No | '' |
| `base-image` | Base image to use for the CVM | No | '' |

## Output Parameters

| Parameter | Description |
|-----------|-------------|
| `cvm-id` | The ID of the created or updated CVM |
| `app-id` | The App ID of the created or updated CVM |
| `cvm-name` | The name used for the CVM (useful when auto-generated from repo name) |
| `deployment-status` | Status of the deployment (success/failed) |
| `deployment-url` | URL to access the deployed application |
| `operation` | The operation performed (create/update) |

## Security Notes

- Store your Phala Cloud API key in GitHub Secrets
- Ensure proper API key permissions when using in public repositories

## Example: Initial Deployment and App ID Storage

This example shows how to create a CVM and store its app-id for future updates:

```yaml
name: Initial Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Phala Cloud
        id: deploy
        uses: Leechael/phala-deploy-action@v2
        with:
          phala-api-key: ${{ secrets.PHALA_CLOUD_API_KEY }}
          cvm-name: 'my-app'
      
      # Store the app-id for future updates
      - name: Save App ID
        if: steps.deploy.outputs.deployment-status == 'success' && steps.deploy.outputs.app-id != ''
        uses: actions/github-script@v6
        with:
          script: |
            await github.rest.actions.createOrUpdateRepoVariable({
              owner: context.repo.owner,
              repo: context.repo.repo,
              name: 'SAVED_APP_ID',
              value: '${{ steps.deploy.outputs.app-id }}'
            })
```

## Example: Update Existing CVM

This example shows how to update an existing CVM using the stored app-id in a subsequent workflow run:

```yaml
name: Update CVM

on:
  push:
    branches: [ main ]

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Update CVM
        uses: Leechael/phala-deploy-action@v2
        with:
          phala-api-key: ${{ secrets.PHALA_CLOUD_API_KEY }}
          app-id: ${{ vars.SAVED_APP_ID }}
          compose-file: './docker-compose.yml'
```

## Example: Conditional Deployment Actions

```yaml
jobs:
  deploy-and-notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Phala Cloud
        id: deploy
        uses: Leechael/phala-deploy-action@v2
        with:
          phala-api-key: ${{ secrets.PHALA_CLOUD_API_KEY }}
      
      - name: Notify Success
        if: steps.deploy.outputs.deployment-status == 'success'
        run: |
          curl -X POST -H "Content-Type: application/json" \
            -d '{"text":"✅ Deployment successful!\nCVM ID: ${{ steps.deploy.outputs.cvm-id }}\nApp ID: ${{ steps.deploy.outputs.app-id }}\nURL: ${{ steps.deploy.outputs.deployment-url }}"}' \
            ${{ secrets.SLACK_WEBHOOK_URL }}
            
      - name: Notify Failure
        if: steps.deploy.outputs.deployment-status == 'failed'
        run: |
          curl -X POST -H "Content-Type: application/json" \
            -d '{"text":"❌ Deployment failed! Please check logs."}' \
            ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Contributing

Contributions are welcome through Pull Requests or Issues.

## License

MIT