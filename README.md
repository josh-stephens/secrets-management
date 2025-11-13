# Secrets Management Skill

**Claude Code skill for age-based secrets management**

A simple, git-versioned approach to managing API keys, tokens, and credentials using modern `age` encryption. Replaces complex solutions like Vault, Infisical, or cloud secret managers with a lightweight CLI wrapper and encrypted files.

## Features

- **Simple encryption** with `age` (https://age-encryption.org/)
- **Git-versioned** encrypted credentials for backup and sync
- **CLI wrapper** for easy secret retrieval
- **Zero runtime overhead** - no containers, no services, no API calls
- **Claude Code integration** - Automatic skill activation when working with secrets
- **Migration guide** from Infisical (Docker-based) to age

## Why age over alternatives?

| Solution | Complexity | Runtime Resources | Use Case |
|----------|-----------|-------------------|----------|
| **age + git** | Low | 0MB RAM, no services | Small teams, personal use, <100 secrets |
| Infisical | High | ~200MB RAM, 3 containers | Large teams, compliance requirements |
| HashiCorp Vault | Very High | ~100MB+ RAM, service | Enterprise, dynamic secrets |
| AWS/GCP Secrets | Medium | API dependency | Cloud-native apps |
| `pass` (GPG) | Medium | GPG complexity | Personal use, but aging tech |

**Choose age when:** You want file-based, git-versioned secrets without running infrastructure.

## Installation

### 1. Install age

```bash
# Ubuntu/Debian
sudo apt install age

# macOS
brew install age

# Arch Linux
sudo pacman -S age
```

### 2. Generate encryption key

```bash
mkdir -p ~/.age
age-keygen -o ~/.age/key.txt
chmod 600 ~/.age/key.txt
```

### 3. Install the skill

```bash
# User-level (recommended - available across all projects)
cd ~/.claude/skills/
git clone https://github.com/josh-stephens/secrets-management.git

# OR Project-level (available in specific project only)
cd /path/to/your/project
mkdir -p .claude/skills
cd .claude/skills/
git clone https://github.com/josh-stephens/secrets-management.git
```

### 4. Install the CLI wrapper

```bash
cp ~/.claude/skills/secrets-management/secrets ~/.local/bin/
chmod +x ~/.local/bin/secrets
```

### 5. Create your secrets vault

```bash
mkdir -p ~/.ssh/secrets
cd ~/.ssh/secrets

# Create your secrets file
cat > credentials.env <<'EOF'
# API Keys
PLEX_TOKEN=your_token_here
GITHUB_TOKEN=ghp_xxxxxxxxxxxx

# Database
DB_PASSWORD=secure_password

# AWS
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
EOF

# Encrypt it
secrets --encrypt credentials.env

# Initialize git for versioning
git init
echo "credentials.env" > .gitignore
git add credentials.env.age .gitignore
git commit -m "Initial secrets vault"

# Delete plaintext (keep only encrypted)
shred -u credentials.env  # Linux
# OR
rm -P credentials.env     # macOS
```

### 6. Optional: Add to CLAUDE.md

Add to `~/.claude/CLAUDE.md` to document the skill:

```markdown
## Custom Skills

- **secrets-management** (`~/.claude/skills/secrets-management/`) - Manage API keys, tokens, and credentials using age encryption. Automatically activated when discussing secrets, credentials, or the `secrets` command. Includes reference.md (commands) and examples.md (integrations).
```

## Usage

### Basic Commands

```bash
# Get a single secret
secrets PLEX_TOKEN

# List all secret keys
secrets --list

# Export all secrets (KEY=value format)
secrets --export

# Load all secrets as environment variables
eval $(secrets -s)

# Decrypt and view all secrets
secrets --decrypt

# Get help
secrets --help
```

### Using in Scripts

```bash
#!/bin/bash

# Load single secret
PLEX_TOKEN=$(secrets PLEX_TOKEN)
curl -H "X-Plex-Token: $PLEX_TOKEN" $PLEX_URL

# OR load all secrets
eval $(secrets -s)
curl -H "X-Plex-Token: $PLEX_TOKEN" $PLEX_URL
```

### Using in Docker Compose

```yaml
services:
  app:
    image: myapp
    environment:
      - API_KEY=${API_KEY}
    env_file:
      - /dev/stdin

# Run with:
# secrets --export | docker-compose up
```

### Claude Code Integration

When you mention secrets in conversation with Claude Code, the skill automatically activates:

**Example prompts:**
- "Show me how to get my API_KEY"
- "I need to add a new secret"
- "How do I list all my secrets?"
- "Help me use secrets in a bash script"

Claude will guide you through the process using the templates and examples.

## File Structure

```
secrets-management/
├── secrets              # CLI wrapper script
├── SKILL.md            # Claude Code skill definition
├── reference.md        # Comprehensive command documentation
├── examples.md         # Real-world usage patterns
├── MIGRATION.md        # Infisical → age migration guide
├── README.md           # This file
└── LICENSE             # MIT License
```

## Migration from Infisical

Migrated from Infisical (Docker-based secrets management) on 2025-10-29.

**Resources reclaimed:**
- Disk: ~2.5GB (Docker images removed)
- Memory: ~200MB (no containers running)
- CPU: Zero overhead

See [MIGRATION.md](MIGRATION.md) for full migration story and rationale.

## Syncing Secrets Across Devices

**Option 1: Private Git Repository**

```bash
cd ~/.ssh/secrets
git remote add origin git@github.com:YOUR_USERNAME/secrets-vault.git
git push -u origin main
```

On other devices:
```bash
git clone git@github.com:YOUR_USERNAME/secrets-vault.git ~/.ssh/secrets
```

**Option 2: Tailscale File Sharing** (if using Tailscale)

```bash
# From device A
cd ~/.ssh/secrets
python3 -m http.server 8000

# From device B (on Tailscale network)
curl http://device-a:8000/credentials.env.age > ~/.ssh/secrets/credentials.env.age
```

**Option 3: SCP/Rsync**

```bash
scp ~/.ssh/secrets/credentials.env.age user@remote:~/.ssh/secrets/
```

## Security Considerations

- **Age key** (`~/.age/key.txt`): Keep private, never commit to git
- **Encrypted file** (`credentials.env.age`): Safe to commit and share
- **Plaintext secrets**: Never commit, delete after encrypting
- **Git repository**: Can be public for encrypted files (key is separate)
- **Backup age key**: Store securely (password manager, printed paper in safe)

## Advantages Over Infisical/Vault

✅ **No runtime overhead** - Just files on disk
✅ **Git-versioned** - Built-in history and sync
✅ **Portable** - Works on any system with `age` installed
✅ **Simple** - 5-minute setup vs hours for container-based solutions
✅ **Offline-first** - No API dependencies or network calls
✅ **Transparent** - Just encrypted files, no databases

❌ **Not suitable for:**
- Teams requiring audit logs and access controls
- Dynamic secrets (database credentials that rotate)
- Compliance requirements (SOC2, HIPAA)
- Large teams (>20 people)
- Secrets shared across many services with complex ACLs

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test with Claude Code
4. Submit a pull request

## Resources

- **GitHub Repository:** https://github.com/josh-stephens/secrets-management
- **Claude Code Skills:** https://docs.claude.com/en/docs/claude-code/skills
- **Age Encryption:** https://age-encryption.org/
- **Age GitHub:** https://github.com/FiloSottile/age

## License

MIT License - See [LICENSE](LICENSE) file

## Version History

**v1.0.0** (2025-10-29)
- Initial release
- `secrets` CLI wrapper
- Claude Code skill integration
- Infisical migration guide
- Comprehensive documentation

---

*Simple secrets management without the complexity*
