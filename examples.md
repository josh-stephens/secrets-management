# Secrets Management - Examples and Workflows

Practical examples for integrating age-encrypted secrets into your applications and workflows.

## Table of Contents

- [Integration Examples](#integration-examples)
  - [Docker Compose](#docker-compose)
  - [Shell Scripts](#shell-scripts)
  - [Python Scripts](#python-scripts)
  - [Node.js Applications](#nodejs-applications)
  - [Systemd Services](#systemd-services)
- [Common Workflows](#common-workflows)
  - [Adding a New Secret](#adding-a-new-secret)
  - [Rotating an API Key](#rotating-an-api-key)
  - [Backing Up Keys](#backing-up-keys)
  - [Setting Up New Team Member](#setting-up-new-team-member)
- [Advanced Use Cases](#advanced-use-cases)
  - [Environment-Specific Secrets](#environment-specific-secrets)
  - [Temporary Secret Sharing](#temporary-secret-sharing)
  - [CI/CD Integration](#cicd-integration)

## Integration Examples

### Docker Compose

**Method 1: Load all secrets into environment**

```bash
# Load secrets and start containers
eval $(secrets -s) && docker-compose up -d

# Or in a script
#!/bin/bash
eval $(secrets -s)
docker-compose up -d
```

**Method 2: Pass specific secrets**

```bash
# docker-compose.yml
services:
  app:
    environment:
      - PLEX_TOKEN=${PLEX_TOKEN}
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}

# Start with secrets loaded
eval $(secrets -s) && docker-compose up -d
```

**Method 3: Use .env file (temporary)**

```bash
#!/bin/bash
# Export secrets to temporary .env file
secrets --export > /tmp/.env

# Use with docker-compose
docker-compose --env-file /tmp/.env up -d

# Clean up
rm /tmp/.env
```

### Shell Scripts

**Example 1: Simple script**

```bash
#!/bin/bash
# my-backup-script.sh

# Load all secrets as environment variables
eval $(secrets -s)

# Use them directly
echo "Backing up to $NFS_SERVER"
rsync -avz /data/ "$NFS_SERVER:/backup/"

# Use in Plex API call
curl -H "X-Plex-Token: $PLEX_TOKEN" \
     "$PLEX_URL_SKIPPY/library/sections"
```

**Example 2: Script with specific secrets**

```bash
#!/bin/bash
# deploy-plex.sh

# Get only the secrets you need
PLEX_TOKEN=$(secrets PLEX_TOKEN)
PLEX_URL=$(secrets PLEX_URL_SKIPPY)

# Use them
curl -X POST "$PLEX_URL/library/sections/all/refresh" \
     -H "X-Plex-Token: $PLEX_TOKEN"

echo "✓ Plex library refreshed"
```

**Example 3: Script with error handling**

```bash
#!/bin/bash
set -e

# Function to get secret with error handling
get_secret() {
    local key=$1
    local value=$(secrets "$key" 2>/dev/null)

    if [ -z "$value" ]; then
        echo "Error: Secret '$key' not found" >&2
        return 1
    fi

    echo "$value"
}

# Get secrets
TRAKT_CLIENT_ID=$(get_secret TRAKT_CLIENT_ID)
TRAKT_CLIENT_SECRET=$(get_secret TRAKT_CLIENT_SECRET)

# Use them
curl -X POST https://api.trakt.tv/oauth/token \
     -d "client_id=$TRAKT_CLIENT_ID" \
     -d "client_secret=$TRAKT_CLIENT_SECRET"
```

### Python Scripts

**Example 1: Using subprocess**

```python
#!/usr/bin/env python3
import subprocess
import sys

def get_secret(key):
    """Get a single secret value"""
    try:
        result = subprocess.check_output(['secrets', key],
                                        stderr=subprocess.DEVNULL)
        return result.decode().strip()
    except subprocess.CalledProcessError:
        print(f"Error: Secret '{key}' not found", file=sys.stderr)
        sys.exit(1)

# Get secrets
plex_token = get_secret('PLEX_TOKEN')
plex_url = get_secret('PLEX_URL_SKIPPY')

# Use them
import requests

response = requests.get(
    f"{plex_url}/library/sections",
    headers={"X-Plex-Token": plex_token}
)
print(response.json())
```

**Example 2: Load all secrets into os.environ**

```python
#!/usr/bin/env python3
import subprocess
import os

# Load all secrets into environment
subprocess.run(['bash', '-c', 'eval $(secrets -s) && env'],
               check=True)

# Now access via os.environ
plex_token = os.environ['PLEX_TOKEN']
domain_name = os.environ['DOMAIN_NAME']

print(f"Domain: {domain_name}")
```

**Example 3: Custom secrets loader**

```python
#!/usr/bin/env python3
import subprocess
from typing import Dict

class SecretsManager:
    """Manager for age-encrypted secrets"""

    def __init__(self):
        self._cache = {}

    def get(self, key: str) -> str:
        """Get a secret by key, with caching"""
        if key not in self._cache:
            result = subprocess.check_output(['secrets', key])
            self._cache[key] = result.decode().strip()
        return self._cache[key]

    def get_all(self) -> Dict[str, str]:
        """Get all secrets as dict"""
        result = subprocess.check_output(['secrets', '--export'])
        secrets_dict = {}

        for line in result.decode().splitlines():
            if '=' in line:
                key, value = line.split('=', 1)
                secrets_dict[key] = value

        return secrets_dict

# Usage
secrets = SecretsManager()
plex_token = secrets.get('PLEX_TOKEN')
all_secrets = secrets.get_all()
```

### Node.js Applications

**Example 1: Using child_process**

```javascript
// secrets.js
const { execSync } = require('child_process');

function getSecret(key) {
  try {
    const value = execSync(`secrets ${key}`, {
      encoding: 'utf8',
      stdio: ['pipe', 'pipe', 'ignore']
    });
    return value.trim();
  } catch (error) {
    throw new Error(`Secret '${key}' not found`);
  }
}

// Usage
const plexToken = getSecret('PLEX_TOKEN');
const plexUrl = getSecret('PLEX_URL_SKIPPY');

console.log(`Connecting to Plex at ${plexUrl}`);
```

**Example 2: Load all secrets at startup**

```javascript
// load-secrets.js
const { execSync } = require('child_process');

function loadSecretsToEnv() {
  try {
    const output = execSync('secrets --export', { encoding: 'utf8' });

    output.split('\n').forEach(line => {
      if (line.includes('=')) {
        const [key, value] = line.split('=');
        process.env[key] = value;
      }
    });

    console.log('✓ Secrets loaded');
  } catch (error) {
    console.error('Error loading secrets:', error.message);
    process.exit(1);
  }
}

// Load at startup
loadSecretsToEnv();

// Now use from process.env
const plexToken = process.env.PLEX_TOKEN;
const apiKey = process.env.TRAKT_CLIENT_ID;
```

**Example 3: Express.js middleware**

```javascript
// secrets-middleware.js
const { execSync } = require('child_process');

// Load secrets once at startup
const secrets = {};
try {
  const output = execSync('secrets --export', { encoding: 'utf8' });
  output.split('\n').forEach(line => {
    if (line.includes('=')) {
      const [key, value] = line.split('=');
      secrets[key] = value;
    }
  });
} catch (error) {
  console.error('Failed to load secrets:', error);
  process.exit(1);
}

// Middleware to inject secrets into req
function secretsMiddleware(req, res, next) {
  req.secrets = secrets;
  next();
}

module.exports = { secrets, secretsMiddleware };

// Usage in app.js
const express = require('express');
const { secretsMiddleware } = require('./secrets-middleware');

const app = express();
app.use(secretsMiddleware);

app.get('/api/plex/status', async (req, res) => {
  const { PLEX_TOKEN, PLEX_URL_SKIPPY } = req.secrets;
  // Use secrets...
});
```

### Systemd Services

**Example 1: Load secrets in ExecStartPre**

```ini
# /etc/systemd/system/my-service.service
[Unit]
Description=My Service with Secrets
After=network.target

[Service]
Type=simple
User=josh
WorkingDirectory=/home/josh/app
Environment="PATH=/home/josh/.local/bin:/usr/local/bin:/usr/bin"

# Load secrets before starting
ExecStartPre=/bin/bash -c 'eval $(secrets -s)'

# Start the service
ExecStart=/usr/bin/node /home/josh/app/server.js

Restart=always

[Install]
WantedBy=multi-user.target
```

**Example 2: Use wrapper script**

```bash
# /home/josh/app/start-with-secrets.sh
#!/bin/bash
eval $(secrets -s)
exec /usr/bin/node /home/josh/app/server.js
```

```ini
# /etc/systemd/system/my-service.service
[Unit]
Description=My Service with Secrets
After=network.target

[Service]
Type=simple
User=josh
WorkingDirectory=/home/josh/app
ExecStart=/home/josh/app/start-with-secrets.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

**Example 3: Pass specific secrets as environment**

```bash
# /home/josh/app/get-secrets-env.sh
#!/bin/bash
# Output secrets in systemd EnvironmentFile format
secrets --export | sed 's/^/Environment="/' | sed 's/$/"/'
```

```ini
[Service]
EnvironmentFile=/home/josh/app/secrets.env
ExecStartPre=/home/josh/app/update-secrets-env.sh
ExecStart=/usr/bin/node /home/josh/app/server.js
```

## Common Workflows

### Adding a New Secret

```bash
# Method 1: Using secrets-edit (recommended)
secrets-edit
# Add line: NEW_API_KEY=sk_live_abc123xyz
# Save and exit
# Automatically re-encrypted and committed to git

# Method 2: Manual
age -d -i ~/.age/key.txt ~/.ssh/tailscale-fleet/secrets/credentials.env.age > /tmp/creds.env
echo "NEW_API_KEY=sk_live_abc123xyz" >> /tmp/creds.env
age -e -r $(age-keygen -y ~/.age/key.txt) /tmp/creds.env > ~/.ssh/tailscale-fleet/secrets/credentials.env.age
rm /tmp/creds.env
cd ~/.ssh/tailscale-fleet/secrets
git add credentials.env.age
git commit -m "Added NEW_API_KEY"
git push

# Method 3: Add to template
echo "NEW_API_KEY=" >> ~/.ssh/tailscale-fleet/secrets/credentials.env.template
git add credentials.env.template
git commit -m "Added NEW_API_KEY to template"

# Verify
secrets --list | grep NEW_API_KEY
secrets NEW_API_KEY
```

### Rotating an API Key

```bash
# 1. Get the old key
OLD_KEY=$(secrets API_KEY)
echo "Old key: ${OLD_KEY:0:10}..."

# 2. Generate or obtain new key from service
NEW_KEY="new_api_key_from_service"

# 3. Update in encrypted file
secrets-edit
# Replace: API_KEY=old_value
# With: API_KEY=new_value
# Save and exit

# 4. Test new key
secrets API_KEY
# Verify it returns the new value

# 5. Update applications
eval $(secrets -s)
docker-compose restart app

# 6. Verify service works with new key
curl -H "Authorization: Bearer $(secrets API_KEY)" https://api.example.com/test

# 7. Revoke old key in service dashboard
```

### Backing Up Keys

```bash
# Backup age private key (password-encrypted)
age -p ~/.age/key.txt > ~/age-key-backup.txt.age
# Enter password when prompted

# Store backup securely
scp ~/age-key-backup.txt.age backup-server:/secure/backups/
# Or upload to encrypted cloud storage

# Backup encrypted secrets (safe to store anywhere)
cp ~/.ssh/tailscale-fleet/secrets/credentials.env.age ~/backups/

# Full backup with git
cd ~/.ssh/tailscale-fleet/secrets
git bundle create ~/secrets-backup.bundle --all

# Restore from backup
# Age key:
age -d ~/age-key-backup.txt.age > ~/.age/key.txt
chmod 600 ~/.age/key.txt

# Secrets:
cp ~/backups/credentials.env.age ~/.ssh/tailscale-fleet/secrets/

# From git bundle:
git clone ~/secrets-backup.bundle ~/.ssh/tailscale-fleet/secrets
```

### Setting Up New Team Member

**On skippy (admin):**

```bash
# Get team member's public age key
# They run: age-keygen -y ~/.age/key.txt
# They send you: age1abc123xyz...

TEAM_MEMBER_PUBKEY="age1abc123xyz..."

# Re-encrypt secrets for multiple recipients
cd ~/.ssh/tailscale-fleet/secrets
age -d -i ~/.age/key.txt credentials.env.age > /tmp/creds.env

age -e \
  -r $(age-keygen -y ~/.age/key.txt) \
  -r $TEAM_MEMBER_PUBKEY \
  /tmp/creds.env > credentials.env.age

rm /tmp/creds.env

git add credentials.env.age
git commit -m "Added team member to encryption recipients"
git push
```

**On team member's machine:**

```bash
# 1. Generate age key
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt

# Send public key to admin
age-keygen -y ~/.age/key.txt

# 2. After admin re-encrypts, clone repo
git clone git@github.com:org/secrets-private.git \
    ~/.ssh/tailscale-fleet/secrets

# 3. Install secrets wrapper
curl -o ~/.local/bin/secrets https://raw.githubusercontent.com/org/repo/scripts/secrets
chmod +x ~/.local/bin/secrets

# 4. Test
secrets --list
secrets DOMAIN_NAME
```

## Advanced Use Cases

### Environment-Specific Secrets

```bash
# Create separate encrypted files for each environment
~/.ssh/tailscale-fleet/secrets/
├── credentials-production.env.age
├── credentials-staging.env.age
└── credentials-development.env.age

# Modify secrets wrapper to accept environment
secrets -e production PLEX_TOKEN
secrets -e staging PLEX_TOKEN

# Or use environment variable
export SECRETS_ENV=production
secrets PLEX_TOKEN
```

### Temporary Secret Sharing

**Share a single secret with team member temporarily:**

```bash
# On your machine
secrets TEMP_API_KEY | age -p > temp-secret.age
# Enter password when prompted

# Send temp-secret.age to team member (via Slack, email, etc.)

# Team member decrypts
age -d temp-secret.age
# Enters password

# After they use it, rotate the secret
```

### CI/CD Integration

**GitHub Actions example:**

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Install age
        run: |
          sudo apt-get update
          sudo apt-get install -y age

      - name: Setup secrets
        env:
          AGE_SECRET_KEY: ${{ secrets.AGE_SECRET_KEY }}
        run: |
          mkdir -p ~/.age
          echo "$AGE_SECRET_KEY" > ~/.age/key.txt
          chmod 600 ~/.age/key.txt

          # Decrypt secrets
          age -d -i ~/.age/key.txt secrets/credentials.env.age > .env

      - name: Deploy
        run: |
          source .env
          ./deploy.sh

      - name: Cleanup
        if: always()
        run: rm -f .env
```

**GitLab CI example:**

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  before_script:
    - apt-get update && apt-get install -y age
    - mkdir -p ~/.age
    - echo "$AGE_SECRET_KEY" > ~/.age/key.txt
    - chmod 600 ~/.age/key.txt
    - age -d -i ~/.age/key.txt secrets/credentials.env.age > .env
  script:
    - source .env
    - ./deploy.sh
  after_script:
    - rm -f .env
  variables:
    AGE_SECRET_KEY: $CI_AGE_SECRET_KEY
```

---

*See reference.md for detailed command documentation*
*See SKILL.md for quick reference and overview*
