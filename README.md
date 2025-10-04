# Authelia with Traefik - Docker Compose

Complete authentication and authorization solution for protecting your applications with Traefik reverse proxy.

## Features

- **Single Sign-On (SSO)** across all protected applications
- **Two-Factor Authentication (2FA)** with TOTP
- **Session management** with Redis
- **Access control** rules per domain
- **Automatic HTTPS** with Let's Encrypt via Traefik
- **Rate limiting** and brute-force protection

## Prerequisites

- Docker and Docker Compose
- Traefik reverse proxy running with:
  - `dokploy-network` network
  - Let's Encrypt certificate resolver named `letsencrypt`
- Domain with DNS configured

## Quick Start

### 1. Generate Secrets

Generate three random secrets for JWT, session, and storage encryption:

```bash
openssl rand -hex 32  # For AUTHELIA_JWT_SECRET
openssl rand -hex 32  # For AUTHELIA_SESSION_SECRET
openssl rand -hex 32  # For AUTHELIA_STORAGE_ENCRYPTION_KEY
```

### 2. Configure Environment

Copy `.env.example` to `.env` and update with your values:

```bash
AUTHELIA_DOMAIN=auth.yourdomain.com
AUTHELIA_JWT_SECRET=<generated-secret-1>
AUTHELIA_SESSION_SECRET=<generated-secret-2>
AUTHELIA_STORAGE_ENCRYPTION_KEY=<generated-secret-3>
```

### 3. Update Configuration

Edit `config/configuration.yml`:

- Replace `yourdomain.com` with your actual domain
- Replace `AUTHELIA_DOMAIN_PLACEHOLDER` with your Authelia domain (e.g., `auth.yourdomain.com`)
- Update `domain` under `access_control.rules` to match your apps
- Update `session.domain` to your root domain

### 4. Set Up Users

Create your user database from the template:

```bash
cp config/users_database.yml.example config/users_database.yml
```

Generate a secure password hash:

```bash
docker run authelia/authelia:latest authelia crypto hash generate argon2 --password 'your-secure-password'
```

Edit `config/users_database.yml` and:
- Replace `REPLACE_WITH_YOUR_GENERATED_HASH` with your generated hash
- Update email address
- Update displayname as needed

**IMPORTANT:** Never commit `config/users_database.yml` to version control (it's in .gitignore).

### 5. Deploy

Add DNS A record:
```
auth.yourdomain.com → your-server-ip
```

Deploy with Docker Compose:

```bash
docker compose up -d
```

### 6. Protect Your Applications

Add the Authelia middleware to any service you want to protect:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.myapp.rule=Host(`app.yourdomain.com`)
  - traefik.http.routers.myapp.entrypoints=websecure
  - traefik.http.routers.myapp.tls.certresolver=letsencrypt
  - traefik.http.routers.myapp.middlewares=authelia@docker  # Add this line
  - traefik.http.services.myapp.loadbalancer.server.port=8080
```

## Access Control Rules

Edit `config/configuration.yml` to customize access rules:

```yaml
access_control:
  default_policy: deny
  rules:
    # Public - no auth required
    - domain: "public.yourdomain.com"
      policy: bypass

    # Require login
    - domain: "*.yourdomain.com"
      policy: one_factor

    # Require login + 2FA
    - domain: "admin.yourdomain.com"
      policy: two_factor
```

## Configuration Files

- `docker-compose.yml` - Container definitions
- `config/configuration.yml` - Authelia main configuration
- `config/users_database.yml` - User credentials and groups
- `.env` - Environment variables (secrets)

## Volumes

- `authelia-config` - Authelia configuration and database
- `authelia-redis` - Redis session storage

## Ports

- **9091** - Authelia web interface (via Traefik)

## First Login

1. Navigate to `https://auth.yourdomain.com`
2. Login with username `admin` and the password you set in `users_database.yml`
3. Set up 2FA by scanning QR code with authenticator app
4. You can add more users by editing `config/users_database.yml`

## Troubleshooting

### "Access Denied" when visiting protected app

1. Check Authelia logs: `docker logs authelia`
2. Verify access control rules in `config/configuration.yml`
3. Ensure your domain matches the `session.domain` setting

### 2FA not working

1. Check time sync on server: `timedatectl`
2. Verify TOTP settings in `config/configuration.yml`

### Redis connection errors

1. Check Redis is running: `docker ps | grep redis`
2. Verify Redis hostname in `config/configuration.yml`

## Security Notes

- Change default password immediately
- Use strong, unique secrets for JWT, session, and encryption
- Enable 2FA for all users
- Review access control rules regularly
- Keep Authelia updated

## Documentation

- [Authelia Documentation](https://www.authelia.com/overview/prologue/introduction/)
- [Configuration Reference](https://www.authelia.com/configuration/prologue/introduction/)
- [Traefik Integration](https://www.authelia.com/integration/proxies/traefik/)

## License

Configuration files provided as-is. Authelia is licensed under Apache 2.0.
