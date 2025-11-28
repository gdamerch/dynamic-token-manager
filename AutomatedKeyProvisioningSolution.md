# Objective
Automate the key update process for container synchronization between GitHub and the Edge devices. At present, GitHub access keys for container image syncing are updated manually. The new provisioning system should automatically generate, rotate, and deploy updated keys across all devices, reducing manual intervention and ensuring secure, uninterrupted syncing.

# Overview
This document describes how to build an automated pipeline for provisioning and managing keys for container synchronization. The main focus of the document is of GitHub Container Registry tokens to authenticate the registry to manage and access container images and Tailscale.

# Components
1. GitHub Actions Workflow
    - Automate key rotation and deployment
    - Triggers can be scheduled or event-driven
    - Workflow will:
        - Authenticate with GitHub to manage GHCR tokens
        - Generate new GHCR tokens
        - Rotate Tailscale key for edge devices
        - Deploy updated tokens and keys to edge devices
2. GitHub Container Registry (GHCR)
    - Tokens generated must have minimal required scope
        - i.e. “read:packages” or “write:packages”
    - Old tokens must be invalidated after successful deployment
    - Tokens should be securely stored in GitHub Secrets
3. Tailscale
    - Generate ephemeral Tailscale keys with their API
    - Keys are deployed to edge devices
    - Ensure old keys are revoked to avoid security risks

# Workflow example
Here is an example of the Workflow steps to take:
- Trigger
    - This will be a scheduled cron job using a weekly rotation, but this can be changed
    - The trigger can also include manual actions such as pushing to the repository
- Check repository
    - Fetch workflow script and config
- Generate new GHCR token
- Store token in GitHub Secrets (optional)
- Generate new Tailscale key
- Deploy keys to edge device via SSH
    - Alternatively Ansible or Tailscale SSH can be used instead of SSH
- Restart services
- Revoke old keys

You can also include a logging system via Fluent Bit or in the GitHub Action

# Workflow Diagram
Flowchart of the workflow as described above:
<br>
![<br>Trigger: Scheduled cron job
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;V
<br> GitHub Actions Workflow
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |---------> Check Repo
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |---------> Generate GHCR Token ---------> Store token in GitHub Secrets
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |---------> Generate Tailscale key
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |---------> Deploy keys to edge device ---------> Restart services
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |---------> Revoke old keys
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp; |
<br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;V
<br>Log and audit](Automated_Key_Provisioning_Diagram.png "Flow Chart")

# Example .yaml
An example of what the .yaml file may look like:
```yaml
name: Key Rotation & Deployment

on:
  schedule:
    - cron: "0 0 * * 0" # weekly rotation

jobs:
  rotate-keys:
    runs-on: ubuntu-latest
    env:
      GH_ADMIN_TOKEN: ${{ secrets.GH_ADMIN_TOKEN }}
      TAILSCALE_AUTH_KEY: ${{ secrets.TAILSCALE_AUTH_KEY }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Generate new GHCR token via GitHub API
        id: ghcr
        run: |
          echo "Creating new GHCR token..."
          RESPONSE=$(curl -s -X POST -H "Authorization: token $GH_ADMIN_TOKEN" -H "Accept: application/vnd.github+json" https://api.github.com/orgs/YOUR_ORG/packages/container/YOUR_REPO/access_tokens)
          NEW_TOKEN=$(echo "$RESPONSE" | jq -r '.token')
          echo "NEW_GHCR_TOKEN=$NEW_TOKEN" >> $GITHUB_ENV

      - name: Store GHCR token in GitHub Secrets (optional)
        uses: peter-evans/create-or-update-secret@v2
        with:
          secret-name: GHCR_TOKEN
          secret-value: ${{ env.NEW_GHCR_TOKEN }}

      - name: Generate Tailscale key
        id: tailscale
        run: |
          echo "Generating new Tailscale key..."
          NEW_TS_KEY=$(curl -s -X POST -H "Authorization: Bearer $TAILSCALE_AUTH_KEY" -H "Content-Type: application/json" -d '{"capabilities":["login"],"ephemeral":true}' https://api.tailscale.com/api/v2/tailnet/YOUR_TAILNET_NAME/keys | jq -r '.key')
          echo "NEW_TAILSCALE_KEY=$NEW_TS_KEY" >> $GITHUB_ENV

      - name: Deploy keys to Edge devices
        run: |
          echo "Deploying GHCR token and Tailscale key to devices..."
          ssh user@edge-device "echo '${{ env.NEW_GHCR_TOKEN }}' > /etc/ghcr_token && echo '${{ env.NEW_TAILSCALE_KEY }}' > /etc/tailscale_key && systemctl restart container-service tailscaled"

      - name: Revoke old Tailscale keys (optional)
        run: |
          echo "Revoking old Tailscale keys..."
          curl -X POST -H "Authorization: Bearer $TAILSCALE_AUTH_KEY" https://api.tailscale.com/api/v2/tailnet/YOUR_TAILNET_NAME/keys/revoke-old
```

# References
- [GitHub Docs - "Working with the Container Registry"](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Tailscale API Documentation](https://tailscale.com/api)
    - [Update device key](https://tailscale.com/api#tag/devices/post/device/{deviceId}/key)
- [Tailscale Auth Keys](https://tailscale.com/kb/1085/auth-keys)