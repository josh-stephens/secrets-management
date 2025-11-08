---
name: secrets-management
description: Manage API keys, tokens, passwords, and credentials using age encryption and git versioning. Use when the user needs to store, retrieve, rotate, edit, or sync secrets securely across devices. Includes commands for the secrets CLI wrapper.
license: MIT
allowed-tools: Read, Bash, Grep, Glob, Edit, Write
metadata:
  version: "2.0.0"
  author: "Josh"
  category: "security"
  updated: "2025-10-31"
  keywords: "secrets, credentials, encryption, age, security, passwords, API keys, tokens"
---

# Secrets Management with `age`

## Overview

Lightweight, git-backed secrets management using `age` encryption. Perfect for small teams and personal projects. Secrets are stored encrypted at `~/.ssh/tailscale-fleet/secrets/` and accessed via the `secrets` CLI command.

## When to Use

Use this skill when the user:
- Wants to store/retrieve API keys, tokens, or passwords securely
- Needs to add, update, or rotate credentials
- Wants to list or view secrets
- Needs to sync secrets across Tailscale devices
- Mentions "secrets", "credentials", "API keys", "tokens", "passwords"
- Asks about the `secrets` command or age encryption

## Architecture

**Storage:** `~/.ssh/tailscale-fleet/secrets/`
- `credentials.env.age` - Encrypted secrets (git-tracked)
- `credentials.env.template` - Template showing available keys
- `.git/` - Git repository for versioning

**Encryption:** `age` (https://age-encryption.org/)
- Modern, simple encryption
- Key at `~/.age/key.txt` (600 permissions)
- Each device can have its own key

**CLI:** `~/.local/bin/secrets`
- Simplifies secret retrieval
- Pattern: `secrets KEYNAME` → returns value
- Multiple output formats

**Format:** KEY=value (one per line)

## Quick Reference

### Retrieve Secrets

```bash
# Get single secret
secrets PLEX_TOKEN

# Use in variable
API_KEY=$(secrets TRAKT_CLIENT_ID)

# List all secret keys
secrets --list

# Load all into environment
eval $(secrets -s)
```

### Edit Secrets

```bash
# If secrets-edit helper exists
secrets-edit

# Or manually
age -d -i ~/.age/key.txt ~/.ssh/tailscale-fleet/secrets/credentials.env.age > /tmp/creds.env
nano /tmp/creds.env
age -e -r $(age-keygen -y ~/.age/key.txt) /tmp/creds.env > ~/.ssh/tailscale-fleet/secrets/credentials.env.age
rm /tmp/creds.env
```

See **reference.md** for detailed editing methods and troubleshooting.

### Common Use Cases

```bash
# Use in Docker Compose
eval $(secrets -s) && docker-compose up -d

# Use in shell script
#!/bin/bash
eval $(secrets -s)
curl -H "Authorization: Bearer $API_KEY" https://api.example.com

# Use in Python
import subprocess
token = subprocess.check_output(['secrets', 'PLEX_TOKEN']).decode().strip()
```

See **examples.md** for integration patterns (Python, Node.js, systemd, CI/CD).

## Current Secrets

From template:
- **Plex:** PLEX_TOKEN, PLEX_URL_YGGDRASILL, PLEX_URL_SKIPPY
- **Trakt:** TRAKT_CLIENT_ID, TRAKT_CLIENT_SECRET, TRAKT_USERNAME
- **Infrastructure:** DOMAIN_NAME (711bf.org), NFS_SERVER, SSL_EMAIL, GENERIC_TIMEZONE
- **SSH:** SSH_KEY_FINGERPRINT, SSH_KEY_CREATED

## Best Practices

1. **Never commit plaintext** - Only `.age` files to git
2. **One key per device** - Or use multi-recipient encryption
3. **Backup age keys** - Store `~/.age/key.txt` securely
4. **Use git** - Version control provides audit trail
5. **Rotate regularly** - Update API keys on schedule
6. **Limit permissions** - `chmod 600` on keys
7. **Use the wrapper** - `secrets` command is simpler than raw age
8. **Document** - Update template when adding secrets

## Security Notes

- Age keys more secure than password-based encryption
- Git commits provide audit trail
- Works offline, no external services
- No network exposure (unlike Infisical web UI)
- ~2MB footprint vs Infisical's ~200MB+

## Comparison to Infisical

We migrated from Infisical (2025-10-29) because:

**Advantages:**
- Simpler (no containers/databases)
- Lighter (~2MB vs ~200MB RAM)
- Faster (instant vs API calls)
- Offline capable
- Git-native workflows

**Trade-offs:**
- No web UI (edit files directly)
- No built-in RBAC
- Manual editing vs forms
- No automatic rotation

**Infisical better for:**
- Teams 10+ people
- Granular RBAC needed
- Multiple projects/environments
- Compliance requirements
- Service-to-service auth

For our Tailscale fleet with ~15 secrets, age + git is perfect.

## Related Resources

- **reference.md** - Detailed command reference, troubleshooting, installation guide
- **examples.md** - Integration patterns, workflows, Docker/Python/Node.js examples
- **MIGRATION.md** - History of migration from Infisical (in archive/)
- [Tailscale Fleet](~/.claude/docs/tailscale-fleet.md) - Device inventory where secrets are used
- [Tailscale SSH Setup](~/.claude/docs/tailscale-ssh-setup.md) - SSH access for remote deployment
- [Linux Management](~/.claude/skills/linux-management/SKILL.md) - Managing Linux devices
- [Mac Management](~/.claude/skills/mac-management/SKILL.md) - Managing Mac devices
- [Windows Management](~/.claude/skills/windows-management/SKILL.md) - Managing Windows devices

## Commands Summary

```bash
# Retrieval
secrets KEYNAME              # Get single secret
secrets --list               # List all keys
secrets --export             # Export all as KEY=value
secrets -s                   # For eval: eval $(secrets -s)
secrets --decrypt            # Show full file

# Management
secrets --encrypt FILE       # Encrypt a file
secrets-edit                 # Edit (if helper exists)

# Git operations
cd ~/.ssh/tailscale-fleet/secrets
git add credentials.env.age
git commit -m "Updated secrets"
git push
```

---

For detailed documentation, see:
- **reference.md** - Full command reference and troubleshooting
- **examples.md** - Integration patterns and workflows
