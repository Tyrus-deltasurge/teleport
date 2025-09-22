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

### Infrastructure Setup

I'll use Terraform with the Auth0 provider to manage the tenant configuration. Here's the basic setup:

```hcl
provider "auth0" {
  domain        = var.auth0_domain  # Pull from GitHub Secrets
  client_id     = var.auth0_client_id
  client_secret = var.auth0_client_secret
}
```

For security, I'm going with WebAuthn as the primary MFA method - it's phishing-resistant and provides a better user experience than SMS or email codes. TOTP will be available as a fallback:

```hcl
resource "auth0_guardian" {
  policy = "all-applications"  # MFA everywhere, no exceptions
  webauthn_roaming { enabled = true }
  otp { enabled = true }
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
- Generating `grafana.ini` config file with OIDC settings
- Grafana will consume Auth0's OIDC endpoints

The API has a 15 req/sec rate limit, so I'll add delays between operations. 

## Security Considerations

### Authentication
Going with a defense-in-depth approach:
- WebAuthn for phishing resistance (no passwords to steal)
- Enforcing MFA on all applications
- Short session timeouts (8 hours active, 4 hours idle)
- Rotating refresh tokens

### Secrets Management
Everything sensitive goes in GitHub Secrets and (gitignore):
- AUTH0_DOMAIN
- AUTH0_CLIENT_ID  
- AUTH0_CLIENT_SECRET
- AUTH0_API_TOKEN

No credentials in code, ever. I'll also scope tokens to the minimum required permissions for each workflow.

### Attack Protection
Setting up Auth0's built-in protections:
- Brute force: Lock accounts after 5 failed attempts for 15 minutes
- Suspicious IPs: Auto-block for 1 hour
- Anomaly detection: Flag and alert on unusual patterns

## Implementation Details

The user registration script will follow this flow:
```bash
# Pseudo code for clarity
for each user in CSV:
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
- **State conflicts**: Always download the latest state artifact before running Terraform
- **Rate limiting**: Exponential backoff with max 3 retries
- **Partial failures**: Log and continue - don't let one bad user stop the whole batch
- **Concurrent workflows**: Use GitHub's concurrency groups to prevent race conditions

## Trade-offs & TODOs

I'm making some deliberate simplifications for the MVP:

| MVP Approach | Production Approach |
|--------------|-------------------|
| State in GitHub Artifacts | S3 with DynamoDB 
| Manual secret rotation | AWS Secrets Manager |
| No role management | RBAC with groups and permissions |

These will all be marked with TODO comments in the code so they're easy to find and upgrade later.

## Success Criteria

The implementation works if:
- WebAuthn is enabled and users are forced to use it
- Grafana users can log in via OIDC without issues
- New users get their password reset emails
- All three workflows run green
- No credentials appear in any logs

## Questions

A few things I'd like your input on:

1. Is requiring WebAuthn too aggressive for an MVP? Should I allow password + MFA as an option initially?
2. Any specific Auth0 attack protection settings you've found particularly effective?
3. Should the user CSV include any additional metadata fields beyond email and name?

---
