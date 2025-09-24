# teleport
MVP for Teleport
# Auth0 Workflow Automation MVP - Design Document

**Date:** September 2025  
**Author:** Tyrus Rachel III      
**Reviewers:** @russjones @oeric @r0mant  

## Overview

I'm building a GitHub Actions-based solution to automate Auth0 tenant management. The goal is to create a simple but secure MVP that can configure tenants, add applications, and manage users without manual intervention.

## Scope

The MVP will deliver three main workflows:
- Configure an Auth0 tenant with strong security settings using Terraform
- Add web applications (starting with Grafana) with OIDC authentication
- Register users in the Auth0 tenant via Python scripts 

I'm intentionally keeping this simple - no multi-tenant support, no RBACs management, no automated secret rotation (yet), and I'll use GitHub Artifacts for Terraform state instead of setting up an Amazon S3 bucket. These can all be addressed in a production version.




## Technical Approach

### Change Review Process
- All Terraform changes go through `terraform plan` in PR before apply
- Plan output is reviewed in GitHub Actions logs
- Only merged PRs trigger `terraform apply` to main
- Manual approval required for production changes


### Infrastructure Setup

I'll use Terraform with the Auth0 provider to manage the tenant configuration. Here's the basic setup:

```hcl
provider "auth0" {
  domain        = var.auth0_domain  # Pull from GitHub Secrets
  client_id     = var.auth0_client_id
  client_secret = var.auth0_client_secret
}
```

For security, I'll implement WebAuth, a phishing-resistant MFA method that is on Auth0's free tier:

```hcl
resource "auth0_guardian" {
  policy = "all-applications"  # MFA everywhere, no exceptions
  webauthn_roaming { enabled = true }
  otp { enabled = true }
 # SMS and email disabled for non-phishing 
}
```

### GitHub Actions Workflows

I'm structuring this MVP with three separate workflows:

1. **configure-tenant.yml** - Runs Terraform to set up the Auth0 tenant with security policies, MFA, and attack protection. Triggers on pushes to Terraform files or manual dispatch.

2. **add-application.yml** - Adds applications to the tenant. For the MVP, I'm implementing Grafana with OIDC. This workflow will create the Auth0 application, configure callbacks, and output a ready-to-use Grafana config file.

3. **register-users.yml** - Runs a shell script that reads a CSV file and bulk creates users. Each user gets a temporary password and an immediate reset email.

### API Integration

**Auth0 Management API v2** - Primary API for tenant configuration:
- `/api/v2/users` for user creation
- `/dbconnections/change_password` for password resets
- Rate limit: 15 req/sec (will add 1-second delays between operations)

**Auth0 Authentication API** - Used by Grafana for OIDC:
- `/authorize` - Initial auth request
- `/oauth/token` - Token exchange
- `/userinfo` - User profile retrieval

**GitHub Actions** - Workflow orchestration:
- Using built-in actions for Terraform setup (`hashicorp/setup-terraform@v2`)
- Artifacts API for state storage (`actions/upload-artifact@v3`)
- Secrets management via GitHub Secrets context

**OSS Grafana** - Configuration only:
- Not calling Grafana APIs directly
- Grafana will consume Auth0's OIDC endpoints

The API has a 15 req/sec rate limit, so I'll add delays between operations. 

**OSS Grafana** -Docker Compose file with the OIDC settings:
yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - '3000:3000'
    environment:
      - GF_AUTH_GENERIC_OAUTH_ENABLED=true
      - GF_AUTH_GENERIC_OAUTH_NAME=Auth0
      - GF_AUTH_GENERIC_OAUTH_CLIENT_ID=<PLACEHOLDER_CLIENT_ID>
      - GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=<PLACEHOLDER_CLIENT_SECRET>
      - GF_AUTH_GENERIC_OAUTH_SCOPES=openid profile email
      - GF_AUTH_GENERIC_OAUTH_AUTH_URL=https://<PLACEHOLDER_DOMAIN>/authorize
      - GF_AUTH_GENERIC_OAUTH_TOKEN_URL=https://<PLACEHOLDER_DOMAIN>/oauth/token
      - GF_AUTH_GENERIC_OAUTH_API_URL=https://<PLACEHOLDER_DOMAIN>/userinfo
      - `GF_AUTH_GENERIC_OAUTH_ALLOW_SIGN_UP=true

## Security Considerations

### Authentication
Going with a defense-in-depth approach:
- WebAuthn for phishing resistance (no passwords to steal)
- Enforcing MFA on all applications
- Short session timeouts ( 4 hours idle)
- Rotating refresh tokens

### Secrets Management
Everything sensitive goes in GitHub Secrets and (gitignore):
- AUTH0_DOMAIN
- AUTH0_CLIENT_ID  
- AUTH0_CLIENT_SECRET
- AUTH0_API_TOKEN

### Terraform Tenant & App Management

Auth0 Scopes:

- read: tenant_settings, update: tenant_settings (Sessions)

- read: guardian_factors, update: guardian_factors (MFA/WebAuth)

### Grafana App Scopes:

-openid profile email
-read: clients, create: clients, update: clients, delete: clients

### User Registration for script:

read: users, create: users, update: users

### Attack Protection
Setting up Auth0's built-in protections:
- Brute force: Lock accounts after 5 failed attempts for 15 minutes
- Suspicious IPs: Auto-block for 1 hour
- Anomaly detection: Flag and alert on unusual patterns

## Implementation Details

The user registration script will follow this flow:
```bash
# Pseudo code for clarity
```
for each user in CSV:
  - Check if user exists via /users-by-email endpoint
  - If exists: log "User already exists" and skip
  - If new: proceed with creation
  - Generate secure temp password
  - Create user via API
  - Send password reset email
  - Enable MFA requirement
  - Sleep 1 second (rate limiting)

    ```

For Grafana integration, I'll generate a complete config file artifact that includes the OIDC settings, so anyone can just drop it into their Grafana instance and it works.

## Edge Cases & Error Handling

Things that will probably go wrong and how I'll handle them:

- **Duplicate users**: Check if they exist before creating
- **Rate limiting**: Exponential backoff with max 3 retries
- **Concurrent workflows**: Use GitHub's concurrency groups to prevent race conditions

## Trade-offs & TODOs

I'm making some deliberate simplifications for the MVP:

| MVP Approach | 
|--------------|
| Manual secret rotation | 
| No role management | 



## Success Criteria

The implementation works if:
- WebAuthn is enabled and users are forced to use it
- Grafana users can log in via OIDC without issues
- New users get their password reset emails
- All three workflows run green
- No credentials appear in any logs

---
