# CaseFlow Outlook OAuth2 IMAP App-Only Setup

## 1. Entra app registration

Required admin roles:
- Entra Application Administrator or Cloud Application Administrator
- Exchange Administrator for Exchange Online steps

Steps:
1. Go to Entra admin center -> Identity -> Applications -> App registrations -> New registration.
2. Name it something explicit such as `CaseFlow Mailbox Reader`.
3. Choose `Accounts in this organizational directory only` unless you have a strict multi-tenant requirement.
4. Leave redirect URI empty for client-credentials flow.
5. Save the app registration.
6. Copy these values immediately:
   - Application (client) ID
   - Directory (tenant) ID
7. Open `Certificates & secrets` -> `Client secrets` -> `New client secret`.
8. Create a secret with the shortest lifetime your security policy allows.
9. Copy the secret value once. This is the `oauthClientSecret` CaseFlow needs.

## 2. Required API permissions

Add application permissions for Office 365 Exchange Online:
1. App registration -> API permissions -> Add a permission.
2. Select `APIs my organization uses`.
3. Search for `Office 365 Exchange Online`.
4. Choose `Application permissions`.
5. Add:
   - `IMAP.AccessAsApp`
   - `SMTP.SendAsApp` only if CaseFlow must send mail through Microsoft SMTP AUTH as the app

Notes:
- Do not use delegated permissions for this scenario.
- Client credentials + XOAUTH2 needs application permissions.

## 3. Admin consent

Steps:
1. Stay on `API permissions`.
2. Click `Grant admin consent for <tenant>`.
3. Confirm the action.
4. Verify both permissions show `Granted for <tenant>`.

## 4. Exchange Online service principal and mailbox authorization

Required admin role:
- Exchange Administrator

Connect to Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
```

Create or verify the Exchange service principal for the Entra app:

```powershell
$appId = '<client-id>'
$sp = Get-ServicePrincipal | Where-Object { $_.AppId -eq $appId }

if (-not $sp) {
  New-ServicePrincipal -AppId $appId -DisplayName 'CaseFlow Mailbox Reader'
}
```

Grant mailbox access for app-only IMAP:

```powershell
$mailbox = 'shared-or-user@contoso.com'
$sp = Get-ServicePrincipal | Where-Object { $_.AppId -eq '<client-id>' }

Add-MailboxPermission -Identity $mailbox -User $sp.Identity -AccessRights FullAccess -InheritanceType All
```

Grant send permission only if SMTP send is needed:

```powershell
Add-RecipientPermission -Identity $mailbox -Trustee $sp.Identity -AccessRights SendAs
```

Important:
- `IMAP.AccessAsApp` alone is not enough. Exchange mailbox authorization is still required.
- App-only access is mailbox-scoped. Repeat the permission step for every mailbox CaseFlow should access.

## 5. Shared mailbox extra steps

For shared mailboxes:
1. Confirm the shared mailbox exists:

```powershell
Get-ExoMailbox -Identity 'shared@contoso.com'
```

2. Apply the same `Add-MailboxPermission` step to the shared mailbox.
3. If outbound SMTP is needed, also apply `Add-RecipientPermission` with `SendAs`.
4. In CaseFlow, use the shared mailbox address in the mailbox `address` field.
5. Use the mailbox address as `imapUsername` unless your backend explicitly documents a different XOAUTH2 username convention.

## 6. Verification commands

Get a token with client credentials:

```bash
curl -X POST "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=<client-id>&client_secret=<client-secret>&scope=https%3A%2F%2Foutlook.office365.com%2F.default&grant_type=client_credentials"
```

Token endpoint:
- `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`

Scope value:
- `https://outlook.office365.com/.default`

Exchange checks:

```powershell
Get-ServicePrincipal | Where-Object { $_.AppId -eq '<client-id>' } | Format-List DisplayName,AppId,Identity
Get-MailboxPermission -Identity 'shared-or-user@contoso.com'
Get-RecipientPermission -Identity 'shared-or-user@contoso.com'
```

Connection values for Outlook / Microsoft 365:
- IMAP host: `outlook.office365.com`
- IMAP port: `993`
- IMAP security: `SSL/TLS`
- SMTP host: `smtp.office365.com`
- SMTP port: `587`
- SMTP security: `STARTTLS`

## 7. Frequent mistakes

- Granting delegated permissions instead of application permissions.
- Forgetting admin consent after adding `IMAP.AccessAsApp` or `SMTP.SendAsApp`.
- Creating the Entra app but not creating or verifying the Exchange Online service principal.
- Skipping mailbox-level authorization. The app can have tenant permission and still fail on a mailbox.
- Using the wrong token scope. Use `https://outlook.office365.com/.default`.
- Testing a shared mailbox without granting `FullAccess` to the app service principal.
- Expecting SMTP send to work without `SMTP.SendAsApp` plus Exchange `SendAs` permission.
- Letting the client secret expire without rotating it in CaseFlow.

## 8. Values to enter in CaseFlow

Use these values in the Outlook mailbox form:
- `tenantId` -> Entra `Directory (tenant) ID`
- `clientId` -> Entra `Application (client) ID`
- `secret` -> client secret value from `Certificates & secrets`

Map them to CaseFlow fields:
- `oauthTenantId` = tenant ID
- `oauthClientId` = client ID
- `oauthClientSecret` = client secret value
- `mailProvider` = `OUTLOOK`
- `authType` = `OAUTH2`

Security recommendations:
- Restrict who can read or rotate the client secret.
- Prefer short-lived secrets and a documented rotation runbook.
- Limit mailbox assignments to only the mailboxes CaseFlow must access.
- Audit `Add-MailboxPermission` and `Add-RecipientPermission` usage regularly.
- Store the secret only in your backend secret store and never expose it in list or detail screens.