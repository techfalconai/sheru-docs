# Microsoft Entra ID SSO Runbook — Sheru Console + Zendesk Portal

One IdP (Entra ID), two independent federations. SSO covers **browser
login by humans**; the Zendesk publish pipeline (`zd_client/`) still needs
its API token regardless of SSO state.

## Prerequisites

- Entra ID tenant GUID (Directory ID). Find it at Entra admin center →
  Identity → Overview → Tenant ID. Must be the GUID — never `common`.
- Sheru console URL (e.g. `https://sheru.epicsec.ca` or the ALB DNS name).
- Zendesk admin access to `https://epicit.zendesk.com/admin`.
- A break-glass Zendesk password admin that is NOT federated (keep it).

## Part 1 — Entra App Registration for Sheru (OIDC)

1. Entra admin center → Identity → Applications → App registrations →
   New registration. Name: `Sheru Console`. Supported account types:
   **Accounts in this organizational directory only** (single tenant).
2. Redirect URI → Web → `https://<sheru-console>/sso-callback`
   (must match the console's `SsoCallback` route exactly, including scheme).
3. Note the **Application (client) ID** and **Directory (tenant) ID**.
4. Certificates & secrets → New client secret (24 months max, calendar the
   rotation). Copy the **Value** immediately — it is shown once.
5. API permissions → keep `openid`, `email`, `profile` (delegated,
   Microsoft Graph `User.Read` is sufficient; no admin consent needed for
   these three). Do NOT add directory-wide scopes.
6. Token configuration → add optional claims `email`, `preferred_username`
   (ID token) so Sheru can route `harvinder@epicsec.ca`-style logins.

## Part 2 — Sheru Console SSO Setup

`PUT /sso-providers?tenant_id=<tenant-uuid>` with (field names match
`SsoProviderIn` in `routes_sso.py` one-to-one):

```json
{
  "issuer": "https://login.microsoftonline.com/<entra-tenant-guid>/v2.0",
  "client_id": "<application-client-id-from-part-1>",
  "client_secret": "<client-secret-value-from-part-1>",
  "default_role": "tenant_viewer",
  "enabled": true,
  "email_domains": ["epicsec.ca"]
}
```

The helper `auth/entra.py::build_provider_payload()` generates exactly
this body (normalizes/strips domains, rejects `common`). Verify routing:

- `GET /auth/sso/discover?email=harvinder@epicsec.ca` must list the tenant
  with `"protocol": "oidc"` (requires either a verified `DomainClaim`
  for `epicsec.ca` or a pre-existing `User` row on that domain — see
  `services/sso_discover.py`).

## Part 3 — Zendesk Enterprise App in Entra (SAML)

1. Entra admin center → Identity → Applications → Enterprise applications →
   New application → search **Zendesk** (gallery app) → Create.
2. Single sign-on → **SAML**. Enter:
   - Identifier (Entity ID): `https://epicit.zendesk.com`
   - Reply URL (ACS): `https://epicit.zendesk.com/access/saml`
   - Sign on URL: `https://epicit.zendesk.com/access/saml`
3. Attributes & Claims → Name identifier: `user.mail`, format
   `emailAddress`. Add fallback claim mapping `user.userprincipalname`
   if any agents lack a populated `mail` attribute.
4. SAML Certificates → download **Certificate (Base64)**; copy the
   **Login URL**.
5. Assign users/groups: the support group only (least privilege — do not
   assign All Users).

## Part 4 — Zendesk Admin Side

1. `https://epicit.zendesk.com/admin` → Security → Team member
   authentication → enable **External authentication → SAML**.
2. Paste the Entra **Login URL** as SSO URL and the **Certificate
   (Base64)** contents into the certificate field.
3. Confirm the ACS URL shown on that page matches
   `https://epicit.zendesk.com/access/saml` before saving.
4. Test with one agent account in a private window. Confirm a NEW agent
   provisioned via SAML lands with role `agent` (not admin) and email
   `…@epicsec.ca`.
5. Only then consider disabling Zendesk password auth. Keep the
   break-glass password admin regardless.

## Part 5 — Test Plan, Rollback, Troubleshooting

Test (in order): Entra discovery doc reachable →
`build_provider_payload` output posts cleanly → test login succeeds for
`harvinder@epicsec.ca` → wrong-domain address (`user@gmail.com`) is NOT
offered the tenant → Zendesk test login provisions agent correctly.

Rollback: Sheru — `PATCH /sso-providers` with `"enabled": false`
(password login resumes immediately). Zendesk — toggle team-member auth
back to Zendesk passwords.

| Symptom | Likely cause |
|---|---|
| `discover` returns `[]` for `@epicsec.ca` | No verified `DomainClaim` and no existing `User` on that domain |
| `nonce mismatch` in logs | Stale login tab / cached redirect; retry the login flow once |
| Zendesk `invalid email` on SAML login | `user.mail` empty → check the `userprincipalname` fallback claim |
| Zendesk still shows password login | SSO enabled but not enforced — expected until Part 4 step 5 |
| API publish still 401s | Expected — unrelated to SSO; fix the API token (see `zd_client/`) |
