# Traefik + Portainer Infrastructure Stack

A complete infrastructure foundation providing Traefik reverse proxy with Let's Encrypt SSL and Portainer container management interface.

## Services Included

- **Traefik v3.5.1**: Reverse proxy and load balancer with automatic SSL certificates
- **Portainer CE/EE**: Web-based container management interface
- **Let's Encrypt**: Automatic SSL certificate provisioning via Route53 DNS challenge

## Quick Start

1. **Copy environment template:**
```bash
cp .env.example .env
```

2. **Configure AWS credentials** (Required for Route53 DNS challenge):
```bash
# Edit .env and set your AWS credentials
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
AWS_REGION=your-region
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
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: AWS credentials for DNS challenge
- `TRAEFIK_HOST` / `PORTAINER_HOST`: Your public domain names
- `LETSENCRYPT_EMAIL`: Required for Let's Encrypt certificate registration
- `PORTAINER_EDITION`: Choose between `ce` (Community) or `ee` (Enterprise)

## Access

After deployment, your services will be available at:

- **Traefik Dashboard**: https://traefik.yourdomain.com
- **Portainer Interface**: https://portainer.yourdomain.com

Both services automatically receive SSL certificates via Let's Encrypt.

## Security Notes

⚠️ **Important Security Considerations:**

1. **Never commit AWS credentials** to version control
2. **Use IAM roles** when possible instead of access keys  
3. **Limit IAM permissions** to only Route53 as shown above
4. **Enable MFA** on AWS accounts with these credentials
5. **Rotate credentials** regularly

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

## Data Persistence

- **Portainer data**: Stored in `portainer_data` volume
- **SSL certificates**: Stored in `traefik_letsencrypt` volume

Both volumes are automatically created and will persist across container restarts.