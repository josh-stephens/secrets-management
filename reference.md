# Secrets Management - Command Reference

Detailed reference for the age-based secrets management system.

## Table of Contents

- [Commands Summary](#commands-summary)
- [Adding and Updating Secrets](#adding-and-updating-secrets)
- [Age Encryption Details](#age-encryption-details)
- [Syncing Across Devices](#syncing-across-devices)
- [Installation on New Device](#installation-on-new-device)
- [Troubleshooting](#troubleshooting)
- [Current Secrets](#current-secrets)

## Commands Summary

### Retrieval Commands

```bash
# Get single secret value
secrets KEYNAME

# List all secret keys (not values)
secrets --list

# Export all secrets as KEY=value format
secrets --export

# Output for sourcing into shell
secrets -s                     # Use with: eval $(secrets -s)

# Show full decrypted file content
secrets --decrypt
```

### Management Commands

```bash
# Encrypt a plaintext file
secrets --encrypt FILE         # Creates FILE.age

# Edit secrets (if secrets-edit helper exists)
secrets-edit

# Manual decryption
age -d -i ~/.age/key.txt FILE.age

# Manual encryption
age -e -r $(age-keygen -y ~/.age/key.txt) FILE > FILE.age
```

### Git Operations

```bash
cd ~/.ssh/tailscale-fleet/secrets

# Check status
git status

# Commit changes
git add credentials.env.age
git commit -m "Updated API keys"

# Push to remote
git push

# Pull updates
git pull
```

## Adding and Updating Secrets

### Method 1: Using secrets-edit Helper

If `~/.local/bin/secrets-edit` exists:

```bash
secrets-edit
```

This will:
1. Decrypt the file to a temporary location
2. Open in your default editor ($EDITOR or nano)
3. Re-encrypt automatically when you save and exit
4. Optionally commit to git

### Method 2: Manual Process

```bash
# Step 1: Decrypt to temporary file
age -d -i ~/.age/key.txt \
    ~/.ssh/tailscale-fleet/secrets/credentials.env.age \
    > /tmp/creds.env

# Step 2: Edit the file
nano /tmp/creds.env
# Add or modify lines in KEY=value format:
# NEW_API_KEY=your_value_here
# EXISTING_KEY=updated_value

# Step 3: Re-encrypt
age -e -r $(age-keygen -y ~/.age/key.txt) \
    /tmp/creds.env \
    > ~/.ssh/tailscale-fleet/secrets/credentials.env.age

# Step 4: Clean up plaintext (IMPORTANT!)
rm /tmp/creds.env

# Step 5: Commit to git
cd ~/.ssh/tailscale-fleet/secrets
git add credentials.env.age
git commit -m "Added NEW_API_KEY and updated EXISTING_KEY"
git push  # if using remote
```

### Method 3: Create secrets-edit Script

If the helper doesn't exist, create it:

```bash
cat > ~/.local/bin/secrets-edit << 'EOF'
#!/bin/bash
set -e

SECRETS_DIR="$HOME/.ssh/tailscale-fleet/secrets"
ENCRYPTED="$SECRETS_DIR/credentials.env.age"
TEMP_FILE=$(mktemp)

echo "Decrypting secrets..."
age -d -i ~/.age/key.txt "$ENCRYPTED" > "$TEMP_FILE"

echo "Opening editor..."
${EDITOR:-nano} "$TEMP_FILE"

echo "Re-encrypting..."
AGE_PUB=$(age-keygen -y ~/.age/key.txt)
age -e -r "$AGE_PUB" "$TEMP_FILE" > "$ENCRYPTED"
rm "$TEMP_FILE"

echo "✓ Secrets updated successfully!"

# Optional: Auto-commit
cd "$SECRETS_DIR"
if git rev-parse --git-dir > /dev/null 2>&1; then
    git add credentials.env.age
    if git commit -m "Updated secrets - $(date +%Y-%m-%d)" 2>/dev/null; then
        echo "✓ Changes committed to git"
    fi
fi
EOF

chmod +x ~/.local/bin/secrets-edit
```

## Age Encryption Details

### Key Generation

```bash
# Generate new age key
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt

# View public key (for multi-recipient encryption)
age-keygen -y ~/.age/key.txt
```

### Single-Recipient Encryption

```bash
# Encrypt for yourself
age -e -r $(age-keygen -y ~/.age/key.txt) input.txt > input.txt.age

# Decrypt
age -d -i ~/.age/key.txt input.txt.age > input.txt
```

### Multi-Recipient Encryption

Encrypt for multiple devices at once:

```bash
# Get public keys from each device
# Device 1 (skippy):
age-keygen -y ~/.age/key.txt
# Output: age1plc44hpe7q6v88jj9wqaqzr6cjkttmu4e4ke9npunec29xtc9e2ssvta3m

# Device 2 (bishop):
ssh bishop "age-keygen -y ~/.age/key.txt"
# Output: age1bishop_public_key_here

# Device 3 (yggdrasill):
ssh yggdrasill "age-keygen -y ~/.age/key.txt"
# Output: age1yggdrasill_public_key_here

# Encrypt for all three devices
age -e \
  -r age1plc44hpe7q6v88jj9wqaqzr6cjkttmu4e4ke9npunec29xtc9e2ssvta3m \
  -r age1bishop_public_key_here \
  -r age1yggdrasill_public_key_here \
  ~/.ssh/tailscale-fleet/secrets/credentials.env \
  > credentials.env.age
```

Each device can decrypt with its own private key.

### Key Backup

```bash
# Backup age key (encrypt with password)
age -p ~/.age/key.txt > ~/age-key-backup.txt.age

# Restore
age -d ~/age-key-backup.txt.age > ~/.age/key.txt
chmod 600 ~/.age/key.txt
```

## Syncing Across Devices

### Method 1: Git (Recommended)

**On primary device (skippy):**

```bash
cd ~/.ssh/tailscale-fleet/secrets

# Initialize git if not already done
git init
git add .
git commit -m "Initial commit"

# Add remote (private repo recommended)
git remote add origin git@github.com:username/secrets-private.git
git push -u origin master
```

**On other devices:**

```bash
# Install age
sudo apt-get install -y age  # Debian/Ubuntu
brew install age              # macOS
# Windows: download from https://github.com/FiloSottile/age/releases

# Generate device-specific age key
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt

# Clone secrets repo
git clone git@github.com:username/secrets-private.git \
    ~/.ssh/tailscale-fleet/secrets

# Install secrets wrapper
curl -o ~/.local/bin/secrets \
    https://raw.githubusercontent.com/username/repo/main/scripts/secrets
chmod +x ~/.local/bin/secrets

# Option A: Copy skippy's key (shared decryption)
scp skippy:~/.age/key.txt ~/.age/
chmod 600 ~/.age/key.txt

# Option B: Re-encrypt for new device's key (separate keys)
# Get your new device's public key
age-keygen -y ~/.age/key.txt
# Then on skippy, re-encrypt with multi-recipient encryption
```

### Method 2: Direct Copy (SCP)

```bash
# Copy encrypted secrets file
scp skippy:~/.ssh/tailscale-fleet/secrets/credentials.env.age \
    ~/.ssh/tailscale-fleet/secrets/

# Copy skippy's age key (easiest - shared key)
scp skippy:~/.age/key.txt ~/.age/
chmod 600 ~/.age/key.txt

# Or generate new key and re-encrypt (separate keys per device)
age-keygen -o ~/.age/key.txt
# Then decrypt on skippy, re-encrypt for new key
```

### Method 3: Tailscale Serve (Temporary Share)

```bash
# On skippy (share temporarily)
cd ~/.ssh/tailscale-fleet/secrets
python3 -m http.server 8000

# On other device
curl http://skippy:8000/credentials.env.age \
    -o ~/.ssh/tailscale-fleet/secrets/credentials.env.age
```

## Installation on New Device

Complete setup from scratch:

```bash
# 1. Install age
# Debian/Ubuntu:
sudo apt-get update && sudo apt-get install -y age

# macOS:
brew install age

# Windows:
# Download from https://github.com/FiloSottile/age/releases
# Extract age.exe to C:\Windows\System32 or add to PATH

# 2. Generate age key
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt

# Display public key (save for multi-recipient encryption)
age-keygen -y ~/.age/key.txt

# 3. Install secrets wrapper
mkdir -p ~/.local/bin
curl -o ~/.local/bin/secrets \
    https://raw.githubusercontent.com/your-repo/scripts/secrets
chmod +x ~/.local/bin/secrets

# Add to PATH if needed
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 4. Get encrypted secrets
mkdir -p ~/.ssh/tailscale-fleet/secrets

# Option A: Clone from git
git clone git@github.com:username/secrets-private.git \
    ~/.ssh/tailscale-fleet/secrets

# Option B: Copy from another device
scp skippy:~/.ssh/tailscale-fleet/secrets/credentials.env.age \
    ~/.ssh/tailscale-fleet/secrets/

# 5. Get decryption key
# Option A: Copy shared key from skippy
scp skippy:~/.age/key.txt ~/.age/
chmod 600 ~/.age/key.txt

# Option B: Re-encrypt for your new key (on skippy)
# ssh skippy "cd ~/.ssh/tailscale-fleet/secrets && age -d -i ~/.age/key.txt credentials.env.age > /tmp/creds.env && age -e -r YOUR_NEW_PUBLIC_KEY /tmp/creds.env > credentials.env.age && rm /tmp/creds.env"

# 6. Test
secrets --list
secrets DOMAIN_NAME
```

## Troubleshooting

### Command Not Found: secrets

**Problem:** `bash: secrets: command not found`

**Solutions:**

```bash
# Check if file exists
ls -la ~/.local/bin/secrets

# Check if directory is in PATH
echo $PATH | grep -o "$HOME/.local/bin"

# Add to PATH if missing
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Or use full path
~/.local/bin/secrets --list
```

### Age Key Not Found

**Problem:** `Error: Age key not found at ~/.age/key.txt`

**Solution:**

```bash
# Generate age key
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt

# Verify
ls -la ~/.age/key.txt
# Should show: -rw------- 1 user user ... ~/.age/key.txt
```

### Encrypted File Not Found

**Problem:** `Error: Encrypted secrets file not found at ~/.ssh/tailscale-fleet/secrets/credentials.env.age`

**Solutions:**

```bash
# Create directory
mkdir -p ~/.ssh/tailscale-fleet/secrets

# Copy from another device
scp skippy:~/.ssh/tailscale-fleet/secrets/credentials.env.age \
    ~/.ssh/tailscale-fleet/secrets/

# Or create from template
cd ~/.ssh/tailscale-fleet/secrets
cp credentials.env.template credentials.env
# Edit credentials.env with your values
age -e -r $(age-keygen -y ~/.age/key.txt) credentials.env > credentials.env.age
rm credentials.env  # Remove plaintext
```

### Cannot Decrypt (Wrong Key)

**Problem:** Decryption fails, produces garbage, or "no identity matched" error

**Cause:** File was encrypted with a different age key

**Solutions:**

```bash
# Option 1: Get the correct key from device that encrypted it
scp skippy:~/.age/key.txt ~/.age/
chmod 600 ~/.age/key.txt

# Option 2: Re-encrypt with your key (if you have plaintext)
# On device with correct key:
age -d -i ~/.age/key.txt credentials.env.age > /tmp/creds.env
scp /tmp/creds.env new-device:/tmp/
rm /tmp/creds.env

# On new device:
age -e -r $(age-keygen -y ~/.age/key.txt) /tmp/creds.env > credentials.env.age
rm /tmp/creds.env

# Option 3: Use multi-recipient encryption (best for teams)
# Encrypt for multiple devices at once (see Age Encryption Details above)
```

### Secret Not Found

**Problem:** `Error: Secret 'KEYNAME' not found in credentials file`

**Solutions:**

```bash
# List available secrets (case-sensitive)
secrets --list

# Search for similar names
secrets --decrypt | grep -i keyname

# Check if secret exists in template
cat ~/.ssh/tailscale-fleet/secrets/credentials.env.template | grep -i keyname

# Add the secret if it doesn't exist
secrets-edit
# Add line: KEYNAME=value
```

### Permission Denied

**Problem:** Permission errors when accessing keys or encrypted files

**Solutions:**

```bash
# Fix age key permissions
chmod 600 ~/.age/key.txt

# Fix secrets directory permissions
chmod 700 ~/.ssh/tailscale-fleet/secrets
chmod 600 ~/.ssh/tailscale-fleet/secrets/credentials.env.age

# Fix secrets script permissions
chmod +x ~/.local/bin/secrets
```

## Current Secrets

From the template (`credentials.env.template`):

**Plex Configuration:**
- `PLEX_CLAIM_TOKEN` - Plex claim token for server setup
- `PLEX_TOKEN` - Plex API authentication token
- `PLEX_URL_YGGDRASILL` - http://100.122.220.42:32999 (primary)
- `PLEX_URL_SKIPPY` - http://100.85.160.91:32400 (secondary)

**Trakt API:**
- `TRAKT_CLIENT_ID` - Trakt API application client ID
- `TRAKT_CLIENT_SECRET` - Trakt API application client secret
- `TRAKT_USERNAME` - Trakt account username

**Infrastructure:**
- `DOMAIN_NAME` - 711bf.org (main domain)
- `SSL_EMAIL` - myn.donos@gmail.com (Let's Encrypt notifications)
- `GENERIC_TIMEZONE` - America/Los_Angeles
- `NFS_SERVER` - 100.122.220.42 (yggdrasill NAS IP)

**SSH Tracking:**
- `SSH_KEY_FINGERPRINT` - Current SSH key fingerprint for tracking
- `SSH_KEY_CREATED` - Date when SSH keys were generated

**Optional:**
- `SMB_USERNAME` - SMB/Windows share username
- `SMB_PASSWORD` - SMB/Windows share password

---

*See examples.md for integration workflows and common use cases*
*See SKILL.md for quick reference and overview*
