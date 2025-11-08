# Secrets Management Migration

**Date:** 2025-10-29
**Status:** ✅ Complete - Migrated from Infisical to `age`

## Summary

Migrated from Infisical (Docker-based secrets management) to `age`-based encryption for simplicity and resource efficiency.

## Why We Migrated

Infisical was overengineered for our needs:
- Required 3 Docker containers (Infisical + PostgreSQL + Redis)
- Used ~200MB+ RAM continuously
- Complex setup and maintenance
- Overkill for ~15 secrets across a small Tailscale fleet

## New Solution: `age` + Git

**Simple, lightweight, git-backed secrets management**

- **Location:** `~/.ssh/tailscale-fleet/secrets/`
- **Encryption:** `age` (https://age-encryption.org/)
- **CLI:** `secrets` command (`~/.local/bin/secrets`)
- **Versioning:** Git repository

### Resources Reclaimed

- **Disk space:** ~2.5GB (removed Docker images)
- **Memory:** ~200MB (no more containers)
- **CPU:** Zero overhead (no services running)

## How to Use

```bash
# Get a secret
secrets PLEX_TOKEN

# List all secrets
secrets --list

# Load all as environment variables
eval $(secrets -s)

# Use in scripts
PLEX_TOKEN=$(secrets PLEX_TOKEN)
curl -H "X-Plex-Token: $PLEX_TOKEN" $PLEX_URL
```

## File Structure

### Active (Current System)
```
~/.ssh/tailscale-fleet/secrets/
├── credentials.env.age          # Encrypted secrets (git-tracked)
├── credentials.env.template      # Template
└── README.md                     # Documentation

~/.age/
└── key.txt                       # Your encryption key

~/.local/bin/
└── secrets                       # CLI wrapper script
```

### Inactive (Old Infisical)
```
~/Projects/secrets/
├── docker-compose.yml            # Not used anymore
├── .env                          # Not used anymore
├── generate-env.sh               # Not used anymore
└── CLAUDE.md                     # Historical reference
```

## Cleanup Completed

- ✅ Stopped and removed all Infisical containers
- ✅ Removed Docker volumes (infisical_db_data, infisical_redis_data)
- ✅ Removed Docker images (~2.5GB freed)
- ✅ Uninstalled Infisical CLI
- ✅ Removed speedtest-cli broken repository

## For More Help

```bash
secrets --help                          # CLI help
cat ~/.claude/skills/secrets.md        # Comprehensive guide
```

Or ask Claude:
- "How do I add a new secret?"
- "Show me how to use secrets in a docker-compose file"
- "How do I sync secrets to another device?"

## Migration Notes

The transition was seamless:
1. Installed `age` (~2MB)
2. Generated encryption key (`~/.age/key.txt`)
3. Created `secrets` wrapper command
4. Encrypted existing template
5. Set up git versioning
6. Tore down Infisical completely
7. Updated documentation

**Result:** Same functionality, 100x simpler, zero ongoing resource usage.
