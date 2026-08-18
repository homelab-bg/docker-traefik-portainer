# Traefik + Portainer Infrastructure Stack

A complete infrastructure foundation providing Traefik reverse proxy with Let's Encrypt SSL, Portainer container management, and dnsweaver DNS automation.

## Services Included

- **Traefik v3.7.10+**: Reverse proxy and load balancer with automatic SSL certificates (v3.6.0+ required on Docker 29+ hosts - see `.env.example`)
- **Portainer CE/EE**: Web-based container management interface
- **Let's Encrypt**: Automatic SSL certificate provisioning via Route53 DNS challenge
- **dnsweaver**: Watches Traefik's labels and auto-creates/removes DNS records for anything deployed with a matching `Host()` rule - e.g. an app deployed via Portainer's GitOps feature gets working DNS with no manual step. On by default (`COMPOSE_PROFILES=dnsweaver` in `.env.example`); see [DNS Automation](#dns-automation-dnsweaver) below.

## Quick Start

1. **Copy environment template:**
```bash
cp .env.example .env
```

2. **Provide credentials as secrets files** (not env vars - see [Credentials](#credentials) below):
```bash
mkdir -p secrets
cat > secrets/aws_credentials <<'EOF'
[default]
aws_access_key_id = your-access-key-id
aws_secret_access_key = your-secret-access-key
EOF
# Only needed if using dnsweaver (on by default):
echo -n "your-technitium-api-token" > secrets/technitium_token
```

3. **Configure domains and services:**
```bash
# Edit .env and set your domains
TRAEFIK_HOST=traefik.yourdomain.com
PORTAINER_HOST=portainer.yourdomain.com
LETSENCRYPT_EMAIL=admin@yourdomain.com

# Choose Portainer edition (ce or ee)
PORTAINER_EDITION=ce
```

4. **Deploy the stack:**
```bash
docker compose up -d
```

## Prerequisites

### Required
- Docker & Docker Compose
- **AWS Route53 hosted zone** for your domain
- **AWS IAM credentials** with Route53 permissions

### AWS IAM Permissions
Your AWS credentials need the following Route53 permissions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange",
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": "*"
        }
    ]
}
```

## Configuration

### Environment Variables
See `.env.example` for all available configuration options.

**Critical Settings:**
- `TRAEFIK_HOST` / `PORTAINER_HOST`: Your public domain names
- `LETSENCRYPT_EMAIL`: Required for Let's Encrypt certificate registration
- `PORTAINER_EDITION`: Choose between `ce` (Community) or `ee` (Enterprise)
- `DATA_DIR`: Where persistent data lives - defaults to `./data` (local bind mount); point at shared/durable storage for anything meant to survive a host rebuild

### Credentials

AWS and Technitium credentials are delivered via mounted secrets files under `secrets/` (gitignored), not plain environment variables - plain env vars are visible via `docker inspect`, a mounted file isn't.

- `secrets/aws_credentials` - standard AWS credentials file format (`[default]` / `aws_access_key_id` / `aws_secret_access_key`)
- `secrets/technitium_token` - a Technitium API token, only needed if dnsweaver is enabled (on by default)

`AWS_REGION` / `AWS_HOSTED_ZONE_ID` aren't secret, so those stay as regular `.env` variables.

## Access

After deployment, your services will be available at:

- **Traefik Dashboard**: https://traefik.yourdomain.com
- **Portainer Interface**: https://portainer.yourdomain.com

Both services automatically receive SSL certificates via Let's Encrypt.

## DNS Automation (dnsweaver)

dnsweaver watches Traefik's Docker labels and, for any container whose `Host()` rule matches `DNSWEAVER_DOMAINS`, creates a CNAME onto `TRAEFIK_HOST` in your Technitium zone - plus a `_dnsweaver.<host>` TXT record marking it as dnsweaver-owned. It removes both when the container stops. Records it didn't create (e.g. this stack's own Terraform-managed DNS) are left alone even if the label matches the domain glob, since it only ever touches records it owns.

**Configuration** (`.env`):
- `TECHNITIUM_URL`, `DNSWEAVER_ZONE`, `DNSWEAVER_DOMAINS` (a glob, e.g. `*.yourdomain.com`)
- `secrets/technitium_token` - create a dedicated Technitium user scoped to just `DNSWEAVER_ZONE` (Technitium's permission model supports per-zone ACLs) and mint a token for that user, rather than reusing an admin token

**Backfill behavior** (confirmed live, not just from docs): it reconciles already-running containers on startup - no restart needed for existing services to pick up DNS once dnsweaver joins the stack.

Disable it (e.g. on a fresh bootstrap before `secrets/technitium_token` exists) by blanking `COMPOSE_PROFILES` in `.env` or passing `--profile ""` on the CLI.

## Security Notes

⚠️ **Important Security Considerations:**

1. **Never commit `secrets/` or `.env`** to version control (both gitignored already)
2. **Use IAM roles** when possible instead of access keys
3. **Limit IAM permissions** to only Route53 as shown above
4. **Enable MFA** on AWS accounts with these credentials
5. **Rotate credentials** regularly
6. **Scope the Technitium token** to just the zone dnsweaver manages, not an admin account

## Network Architecture

This stack creates the `traefik` network that other services should join to be accessible through the reverse proxy.

**For other services to integrate:**
```yaml
networks:
  traefik:
    name: traefik
    external: true

services:
  your-service:
    networks:
      - traefik
    labels:
      - traefik.enable=true
      - traefik.http.routers.your-service.rule=Host(`your-service.yourdomain.com`)
```

## Troubleshooting

**SSL Certificate Issues:**
- Verify AWS credentials have Route53 permissions
- Check Let's Encrypt email is valid
- Ensure DNS propagation is complete
- Check Traefik logs: `docker compose logs traefik`

**Access Issues:**
- Verify domains point to your server's public IP
- Check firewall allows ports 80 and 443
- Ensure Docker daemon is running

**DNS not appearing for a new app:**
- Confirm the `dnsweaver` profile is active (`docker compose ps` should list it) - it's on by default but can be disabled, see [DNS Automation](#dns-automation-dnsweaver)
- Check `docker compose logs dnsweaver` - it logs every record it creates, skips, or fails
- Confirm the container's `Host()` rule matches `DNSWEAVER_DOMAINS`

## Data Persistence

- **Portainer data**: `${DATA_DIR}/portainer-data`
- **SSL certificates**: `${DATA_DIR}/letsencrypt`

`DATA_DIR` defaults to `./data` (bind mount, local to the compose project) - override it in `.env` to point at shared/durable storage for anything meant to survive a host rebuild.