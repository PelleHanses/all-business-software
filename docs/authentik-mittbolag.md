# Authentik: Mittbolag identity setup

The Authentik role deploys the organization blueprint at
`ansible/roles/authentik/files/organization.yaml`. The worker mounts it read-only
at `/blueprints`; Authentik discovers and applies blueprints from that directory.

## Organization and initial accounts

The blueprint creates the Authentik brand for `mittbolag.se` and these accounts:

| Name | Username / email | Phone | Groups |
|---|---|---|---|
| Allan Allansson | allan@mittbolag.se | 07012234567 | users, osd, styrelsen |
| Olle Ollsson | olsson@mittbolag.se | 07054435345 | users, osd |
| Greta Gretadotter | greta@mittbolag.se | 0734234234 | users, osd, styrelsen |

The `phone_number` value is stored as a user attribute. The accounts are active
but no passwords are provisioned by Git. Set initial passwords through the
Authentik administration interface, or configure an invitation/password-reset
flow and SMTP before inviting users. Do not store user passwords in this
repository or in the blueprint.

The blueprint is repeatable: it identifies users and groups by stable names and
does not hard-code database IDs. Review the blueprint and take a database backup
before changing group membership in an existing deployment.

## Public URL and DNS

The inventory enables the Authentik Caddy fragment and sets its hostname to
`mittbolag.se`. Before deploying, ensure:

- `mittbolag.se` has an A/AAAA record pointing to the Caddy VM.
- Ports 80/tcp and 443/tcp reach Caddy.
- The `[caddy]` and `[authentik]` inventory groups both contain the intended
  hosts.
- The required Authentik secrets are present in the encrypted Ansible Vault
  file.

If you prefer to publish Authentik at a subdomain such as
`auth.mittbolag.se`, change both `authentik_caddy_hostname` in
`inventory/group_vars/all.yml` and the `domain` identifier in the brand
blueprint to the same hostname. DNS must then point that hostname at Caddy.

## OIDC for applications

Authentik provides OIDC/OAuth2 endpoints, but each relying application should
have its own **OAuth2/OpenID Provider** and **Application** in Authentik. Do not
share one client secret across unrelated applications.

For each service:

1. In Authentik, create an OAuth2/OpenID Provider. Choose a unique name and
   client ID, set the exact redirect/callback URI(s) required by that service,
   and select the required signing algorithm/scopes.
2. Create an Application and associate it with that provider.
3. Give the service its own client ID and client secret. Store the secret in
   that service's encrypted Ansible Vault variables, not in this blueprint.
4. Configure the service with the issuer/discovery URL and endpoints shown by
   the Authentik provider. The standard discovery URL has this form:

   `https://mittbolag.se/application/o/<slug>/.well-known/openid-configuration`

   Replace `<slug>` with the provider slug displayed by Authentik. Use the
   provider's displayed issuer URL if the application expects an issuer rather
   than a discovery URL.
5. Test login, logout, group/role claims and token expiry with a non-admin test
   account before enabling the integration for all users.

OIDC group claims are not automatically guaranteed by merely adding users to
Authentik groups. Configure a scope/property mapping on each provider and map
the claim names and group names expected by the relying application. Restrict
each provider's access with an Authentik policy/group binding where appropriate.

The organization blueprint intentionally does not create generic OIDC providers:
redirect URIs, scopes, claims, signing settings and access policies differ per
service. Add those definitions only when the target service and its exact OIDC
requirements are known.
