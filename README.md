# Secrets Management Skill

**Claude Code Skill for managing encrypted secrets with age**

This is a user-level skill for Claude Code that provides comprehensive guidance on managing API keys, tokens, passwords, and credentials using `age` encryption.

## What This Is

A Claude Code skill (https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to help you:
- Store secrets securely with `age` encryption
- Retrieve secrets for use in scripts and applications
- Add, update, and rotate credentials
- Sync secrets across devices
- Use the `secrets` CLI wrapper effectively

## Installation

This skill is installed as a **user-level skill** in your home directory:

```
~/Projects/secrets/
├── SKILL.md              # Main skill definition (loaded by Claude Code)
├── README.md             # This file
├── MIGRATION.md          # Migration history from Infisical
└── archive/              # Archived Infisical deployment files
```

## How It Works

When you mention secrets, credentials, API keys, or use the `secrets` command in conversation with Claude Code, Claude will automatically load this skill and use it to help you.

**Example prompts that activate this skill:**
- "Show me how to get my PLEX_TOKEN"
- "I need to add a new API key to my secrets"
- "How do I list all my secrets?"
- "Help me sync secrets to another device"

## The Secrets System

This skill documents our `age`-based secrets management:

- **Storage:** `~/.ssh/tailscale-fleet/secrets/credentials.env.age`
- **Encryption:** `age` (modern, simple encryption)
- **CLI:** `secrets` command wrapper at `~/.local/bin/secrets`
- **Versioning:** Git repository

### Quick Commands

```bash
# Get a secret
secrets PLEX_TOKEN

# List all secrets
secrets --list

# Load all as environment variables
eval $(secrets -s)
```

## Migration from Infisical

On 2025-10-29, we migrated from a self-hosted Infisical deployment to this simpler `age`-based approach.

**Why we switched:**
- Infisical required 3 Docker containers (~200MB+ RAM)
- Over-engineered for ~15 secrets across a small team
- `age` + git is simpler, faster, and uses zero runtime resources

See `MIGRATION.md` for full details.

## Archived Files

The `archive/` directory contains our original Infisical deployment files as a historical record.

## Resources

- **Claude Code Skills Docs:** https://docs.claude.com/en/docs/claude-code/skills
- **Age Encryption:** https://age-encryption.org/
- **Secrets wrapper:** `~/.local/bin/secrets`
- **Main vault:** `~/.ssh/tailscale-fleet/secrets/`

## License

MIT
