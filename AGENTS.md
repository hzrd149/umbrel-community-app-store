# Agent Instructions for Umbrel Community App Store Maintenance

This document provides comprehensive instructions for AI agents to maintain, add, and remove apps in this Umbrel community app store repository.

## Repository Structure

This is a personal Umbrel community app store with the following structure:

```
/
├── umbrel-app-store.yml          # App store configuration
├── README.md                      # User-facing documentation
├── AGENTS.md                      # This file - agent instructions
└── <app-id>/                      # One directory per app
    ├── docker-compose.yml         # Docker services definition (REQUIRED)
    ├── umbrel-app.yml            # App manifest (REQUIRED)
    ├── exports.sh                # Environment variables (OPTIONAL)
    ├── icon.svg or icon.png      # App icon (REQUIRED)
    └── hooks/                    # Lifecycle hooks (OPTIONAL)
        ├── pre-start
        ├── post-start
        ├── pre-stop
        └── post-stop
```

## App ID Naming Convention

App IDs in this community app store MUST follow this pattern:
- Format: `hzrd149-<app-name>`
- Only lowercase letters and dashes
- Must be human-readable and recognizable
- Examples: `hzrd149-nostrudel`, `hzrd149-narr`, `hzrd149-blob-box`

## Adding a New App

### Step 1: Prepare the App ID and Directory

1. Choose an app ID following the naming convention: `hzrd149-<app-name>`
2. Create a new directory with the app ID name
3. Ensure the app has a Docker image available (on Docker Hub or other registry)

### Step 2: Create Required Files

#### A. `umbrel-app.yml` (REQUIRED)

This is the app manifest. Create it with the following structure:

```yaml
manifestVersion: 1                    # Use version 1 or 1.1
id: hzrd149-<app-name>               # Must match directory name
category: <category>                  # e.g., social, networking, files, finance
name: <App Display Name>
version: "<version>"                  # App version (quoted string)
tagline: <short description>
icon: https://raw.githubusercontent.com/hzrd149/umbrel-community-app-store/master/<app-id>/icon.svg
description: >-
  <Detailed description of the app>
releaseNotes: |
  <What's new in this version>
developer: <developer name>
website: <app website URL>
dependencies: []                      # Array of app IDs this app depends on
repo: <source code repository URL>
support: <support/issues URL>
port: <port-number>                   # Unique port for this app
gallery:
  - <screenshot URL 1>
  - <screenshot URL 2>
path: ""                              # Path suffix (usually empty)
defaultUsername: ""                   # If app has default credentials
defaultPassword: ""
submitter: hzrd149
submission: ""
```

**Important fields:**
- `port`: Must be unique across all apps in the store (check existing apps)
- `icon`: Should point to the raw GitHub URL of the icon in this repo
- `category`: Choose from: social, networking, files, finance, automation, development, media, bitcoin, lightning
- `dependencies`: List app IDs that must be installed first (e.g., `["bitcoin", "lightning"]`)

#### B. `docker-compose.yml` (REQUIRED)

Create a Docker Compose file with this template:

```yaml
services:
  app_proxy:
    environment:
      # Format: <app-id>_<service-name>_1
      APP_HOST: <app-id>_<service-name>_1
      APP_PORT: <port-number>
      # Optional: Disable authentication
      # PROXY_AUTH_ADD: "false"
      # Optional: Whitelist/blacklist paths
      # PROXY_AUTH_WHITELIST: "/api/*"
      # PROXY_AUTH_BLACKLIST: "/admin/*"

  <service-name>:
    image: <docker-image>:<tag>@sha256:<digest>
    restart: on-failure
    stop_grace_period: 1m
    volumes:
      # Persistent data storage
      - ${APP_DATA_DIR}/data:/data
      # Optional: Mount Bitcoin Core data (read-only)
      # - ${APP_BITCOIN_DATA_DIR}:/bitcoin:ro
      # Optional: Mount Lightning Node data (read-only)
      # - ${APP_LIGHTNING_NODE_DATA_DIR}:/lnd:ro
    environment:
      # App configuration variables
      # Available Umbrel variables:
      # - $DEVICE_HOSTNAME
      # - $DEVICE_DOMAIN_NAME
      # - $TOR_PROXY_IP
      # - $TOR_PROXY_PORT
      # - $APP_HIDDEN_SERVICE
      # - $APP_PASSWORD
      # - $APP_SEED
    networks:
      default:
        ipv4_address: $APP_<APP_NAME>_IP
```

**Key points:**
- Pin Docker images to specific sha256 digests for security
- Use `${APP_DATA_DIR}` for persistent storage
- Use `restart: on-failure` for automatic recovery
- The `app_proxy` service is automatically provided by Umbrel
- Network IP addresses use the pattern `$APP_<APP_NAME>_IP`

#### C. `exports.sh` (OPTIONAL)

Create this file if you need to export environment variables for use in docker-compose.yml or by other apps:

```bash
#!/usr/bin/env bash

export APP_<APP_NAME>_IP="10.21.21.X"  # Choose unique IP in 10.21.21.0/24 range
export APP_<APP_NAME>_PORT="<port>"
```

Make it executable:
```bash
chmod +x exports.sh
```

#### D. Icon File (REQUIRED)

- Add an `icon.svg` (preferred) or `icon.png` file
- Should be 256x256 pixels
- No rounded corners (Umbrel applies rounding via CSS)
- Update the `icon` URL in `umbrel-app.yml` to point to the raw GitHub URL

#### E. Hooks (OPTIONAL)

Create a `hooks/` directory with lifecycle scripts if needed:

- `pre-start`: Runs before app starts
- `post-start`: Runs after app starts
- `pre-stop`: Runs before app stops
- `post-stop`: Runs after app stops

Make hooks executable:
```bash
chmod +x hooks/*
```

**Common Use Case: Data Folder Ownership**

When an app runs as a non-root user (which is a security best practice), it may not have permission to access the `${APP_DATA_DIR}` folder, which is typically owned by root. Use a `pre-start` hook to fix ownership before the app starts.

Example `hooks/pre-start` script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Change ownership of data directory to match the app's user
# Replace 1000:1000 with the UID:GID your app runs as
chown -R 1000:1000 "${APP_DATA_DIR}" || true
```

**Finding the Correct UID:GID:**

To determine what UID:GID your app runs as:

1. Check the Dockerfile for the image (look for `USER` directive)
2. Common non-root users:
   - `1000:1000` - Default user on many systems
   - `node` user - Often UID 1000 in Node.js images
   - `nginx` user - Often UID 101 in nginx images
   - Custom users - Check the image documentation

**More Complex Example:**

For apps that need multiple directories with specific permissions:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Create subdirectories if they don't exist
mkdir -p "${APP_DATA_DIR}/data"
mkdir -p "${APP_DATA_DIR}/config"
mkdir -p "${APP_DATA_DIR}/logs"

# Set ownership (replace with your app's UID:GID)
chown -R 1000:1000 "${APP_DATA_DIR}"

# Set specific permissions if needed
chmod 755 "${APP_DATA_DIR}/data"
chmod 600 "${APP_DATA_DIR}/config"/* 2>/dev/null || true
```

**Important Notes:**

- The `|| true` prevents the script from failing if the directory is already correctly owned
- Use `set -euo pipefail` to catch errors early
- The hook runs as root, so it has permission to change ownership
- Always test that your app can read/write after the ownership change

### Step 3: Validate the Configuration

Before committing, verify:

1. **Port uniqueness**: Check all other apps' `umbrel-app.yml` files to ensure the port is not used
2. **IP address uniqueness**: If using `exports.sh`, ensure the IP is unique
3. **Docker image**: Verify the image exists and is accessible
4. **File permissions**: Ensure hooks and exports.sh are executable
5. **YAML syntax**: Validate YAML files are properly formatted
6. **Icon accessibility**: Verify icon file exists and is committed

### Step 4: Test the App

Testing methods:

1. **Local umbrelOS development environment**: Follow Umbrel's dev setup
2. **Physical device**: Copy to device and test installation

Test checklist:
- [ ] App installs successfully
- [ ] App starts without errors
- [ ] Web UI is accessible at the specified port
- [ ] Persistent data survives app restart
- [ ] App uninstalls cleanly
- [ ] No conflicts with other apps

### Step 5: Commit and Push

```bash
git add <app-id>/
git commit -m "Add <app-name> v<version>"
git push origin master
```

## Updating an Existing App

### When to Update

Update an app when:
- New version is released
- Bug fixes are needed
- Configuration changes are required
- Docker image needs updating

### Update Process

1. **Update the Docker image** (if applicable):
   - Build and push new multi-arch image to Docker Hub
   - Get the new sha256 digest
   - Update `docker-compose.yml` with new image reference

2. **Update `umbrel-app.yml`**:
   - Increment the `version` field
   - Update `releaseNotes` with changes
   - Update any other changed fields

3. **Update other files** as needed:
   - Modify `docker-compose.yml` for configuration changes
   - Update `exports.sh` if environment variables changed
   - Replace icon if it changed

4. **Test the update**:
   - Install the old version
   - Update to the new version
   - Verify data persists and app works correctly

5. **Commit and push**:
   ```bash
   git add <app-id>/
   git commit -m "Update <app-name> to v<version>"
   git push origin master
   ```

## Removing an App

### When to Remove

Remove an app when:
- App is no longer maintained
- App has critical security issues
- App is superseded by another app
- App is incompatible with current Umbrel version

### Removal Process

1. **Check dependencies**: Ensure no other apps depend on this app
   ```bash
   grep -r "dependencies:" */umbrel-app.yml | grep "<app-id>"
   ```

2. **Remove the app directory**:
   ```bash
   rm -rf <app-id>/
   ```

3. **Commit and push**:
   ```bash
   git commit -m "Remove <app-name> - <reason>"
   git push origin master
   ```

## Common Environment Variables

Apps can use these Umbrel-provided environment variables:

### System Variables
- `$DEVICE_HOSTNAME` - Umbrel device hostname (e.g., "umbrel")
- `$DEVICE_DOMAIN_NAME` - Local domain name (e.g., "umbrel.local")

### Tor Proxy Variables
- `$TOR_PROXY_IP` - Local IP of Tor proxy
- `$TOR_PROXY_PORT` - Port of Tor proxy

### App-Specific Variables
- `$APP_HIDDEN_SERVICE` - Tor hidden service address for the app
- `$APP_PASSWORD` - Unique password for app authentication (shown to user)
- `$APP_SEED` - Unique 256-bit hex string derived from user's Umbrel seed
- `$APP_DATA_DIR` - Directory for app's persistent data
- `$APP_BITCOIN_DATA_DIR` - Bitcoin Core data directory (if available)
- `$APP_LIGHTNING_NODE_DATA_DIR` - Lightning node data directory (if available)

## Port Allocation

Current port allocations in this app store:

| App ID | Port | Service |
|--------|------|---------|
| hzrd149-narr | 3395 | Web UI |
| (check other apps) | ... | ... |

**When adding a new app**: Choose a port that doesn't conflict with:
- Other apps in this store
- Official Umbrel apps (typically 3000-4000 range)
- System services

Recommended range for community apps: 3300-3999, 8000-8999

## Troubleshooting Common Issues

### App Won't Start
- Check Docker image is accessible
- Verify port is not in use
- Check volume mount paths are correct
- Review app logs: `umbreld client apps.logs --appId <app-id>`

### Data Not Persisting
- Ensure volumes are mapped to `${APP_DATA_DIR}`
- Verify the app writes to the mounted paths
- Check volume permissions

### Port Conflicts
- Search for port usage: `grep -r "port: <number>" */umbrel-app.yml`
- Choose a different port and update both yml files

### Authentication Issues
- Check `app_proxy` configuration in docker-compose.yml
- Verify `APP_HOST` matches the service name pattern
- Consider using `PROXY_AUTH_ADD: "false"` if app has own auth

## Best Practices

### Docker Images
- ✅ Use official or trusted images
- ✅ Pin to specific sha256 digests
- ✅ Use multi-arch images (arm64 and amd64)
- ✅ Use lightweight base images
- ✅ Use multi-stage builds for smaller images
- ❌ Don't use `:latest` tag
- ❌ Don't run as root user

### Security
- ✅ Use read-only mounts when possible (`:ro`)
- ✅ Limit exposed ports
- ✅ Use Umbrel's app proxy for authentication
- ✅ Validate environment variables
- ❌ Don't expose sensitive data in environment variables
- ❌ Don't disable authentication unless necessary

### Data Management
- ✅ Map all persistent data to volumes
- ✅ Use `${APP_DATA_DIR}` for app data
- ✅ Test data persistence by restarting app
- ❌ Don't store data in container filesystem
- ❌ Don't assume data will persist without volumes

### Configuration
- ✅ Use descriptive app IDs and names
- ✅ Write clear descriptions and release notes
- ✅ Set appropriate categories
- ✅ Include support and repository links
- ✅ Test on both arm64 and amd64 if possible
- ❌ Don't use special characters in app IDs
- ❌ Don't forget to update version numbers

## Reference Documentation

- Official Umbrel App Framework: https://github.com/getumbrel/umbrel-apps/blob/master/README.md
- Docker Compose Documentation: https://docs.docker.com/compose/
- Umbrel GitHub: https://github.com/getumbrel/umbrel
- Community App Store Template: https://github.com/getumbrel/umbrel-community-app-store

**Note for Agents**: Always verify configurations against existing apps in the repository to ensure consistency and avoid conflicts. When in doubt, examine similar existing apps as examples.
