# Lab 02: JML Cycle #2, Admin Password Reset, and RBAC Investigation via Microsoft Graph PowerShell

## Objective

Build on Lab 01 by running a second full Joiner-Mover-Leaver cycle on a separate test user, practicing an admin-initiated password reset, and investigating how Microsoft Entra ID's Role-Based Access Control (RBAC) layer interacts with Microsoft Graph API scopes/permissions.

## Environment

- **Tenant**: Microsoft Entra ID tenant (Azure account)
- **Tools**: Microsoft Graph PowerShell SDK
- **PowerShell version**: 7.6.3 (latest)
- **Auth method**: Delegated, device code flow
- **Scopes used**: `User.ReadWrite.All`, `Group.ReadWrite.All`, `Directory.ReadWrite.All`, `Directory.AccessAsUser.All`

## Part 1: Joiner — Second Test User

Created a second, independent test user (kept separate from Lab 01's `Test Joiner` by using distinct variable names and a distinct UPN, to avoid session/object collisions):

```powershell
$PasswordProfile2 = @{
    Password = "TempP@ssw0rd456!"
    ForceChangePasswordNextSignIn = $true
}

$newUser2 = New-MgUser -DisplayName "Test Joiner 2" `
    -UserPrincipalName "test.joiner2@<tenant-domain>.onmicrosoft.com" `
    -AccountEnabled `
    -MailNickname "testjoiner2" `
    -PasswordProfile $PasswordProfile2 `
    -Department "Sales"
```

Created a new test group for this run:

```powershell
$trainersGroup = New-MgGroup -DisplayName "Test Trainers Team" -MailEnabled:$false -MailNickname "testtrainers" -SecurityEnabled:$true
```

Added the new user to it as their initial access grant:

```powershell
New-MgGroupMember -GroupId $trainersGroup.Id -DirectoryObjectId $newUser2.Id
```

**Verification (checked both directions — group side and user side):**

```powershell
Get-MgGroupMember -GroupId $trainersGroup.Id
Get-MgUserMemberOf -UserId $newUser2.Id
```

Both confirmed Test Joiner 2's membership in Test Trainers Team.

## Part 2: Admin-Initiated Password Reset

Practiced resetting a user's password as an administrator — a common Tier 1/helpdesk IAM task.

```powershell
$NewPasswordProfile = @{
    Password = "NewP@ssw0rd789!"
    ForceChangePasswordNextSignIn = $true
}

Update-MgUser -UserId $newUser2.Id -PasswordProfile $NewPasswordProfile
```

**Initial attempt failed** with a 403 `Authorization_RequestDenied` error, which led to the RBAC investigation covered in the Troubleshooting Appendix below. After reconnecting with the `Directory.AccessAsUser.All` scope added, the reset succeeded.

**Note on verification:** Microsoft Graph does not return plaintext passwords once set — not even to an admin — since passwords are hashed immediately on Microsoft's end. The only confirmation of a successful reset is the absence of an error from `Update-MgUser`, or an actual sign-in test using the new credential.

## Part 3: Mover — Trainers to Marketing

Moved Test Joiner 2 from Trainers into the pre-existing Marketing group (created in Lab 01) and updated their department attribute:

```powershell
$marketingGroup = Get-MgGroup -Filter "displayName eq 'Test Marketing Team'"

Remove-MgGroupMemberByRef -GroupId $trainersGroup.Id -DirectoryObjectId $newUser2.Id
New-MgGroupMember -GroupId $marketingGroup.Id -DirectoryObjectId $newUser2.Id
Update-MgUser -UserId $newUser2.Id -Department "Marketing"
```

**Verification:**

```powershell
Get-MgGroupMember -GroupId $trainersGroup.Id       # Empty — confirms removal
Get-MgGroupMember -GroupId $marketingGroup.Id      # Shows Test Joiner 2 — confirms addition
Get-MgUser -UserId $newUser2.Id -Property DisplayName, Department | Format-List
```

Confirmed `Department: Marketing` and correct group membership.

## Part 4: Leaver — Offboarding

```powershell
# Disable sign-in
Update-MgUser -UserId $newUser2.Id -AccountEnabled:$false

# Remove all group memberships
Get-MgUserMemberOf -UserId $newUser2.Id | ForEach-Object {
    Remove-MgGroupMemberByRef -GroupId $_.Id -DirectoryObjectId $newUser2.Id
}
```

**Verification:**

```powershell
Get-MgUser -UserId $newUser2.Id -Property DisplayName, AccountEnabled | Format-List
Get-MgUserMemberOf -UserId $newUser2.Id
```

Confirmed `AccountEnabled: False` with no remaining group memberships — Test Joiner 2's full JML cycle complete.

## Part 5: RBAC Investigation

Following the password-reset failure, investigated how directory roles (RBAC) sit alongside Graph API scopes as a second, independent permission layer.

**Key distinction identified:**

| Layer | Controls | Example |
|---|---|---|
| Graph API scope/permission | What a *token* is allowed to request | `User.ReadWrite.All` |
| Directory role (RBAC) | What an *account* is actually authorized to do | Global Administrator, User Administrator, Helpdesk Administrator |

Both layers must align for a privileged operation (like resetting another user's password) to succeed — a valid scope alone is not sufficient if the account's assigned role doesn't include that right.

**Checked own role assignment:**

```powershell
$me = Get-MgUser -Filter "userPrincipalName eq 'admin@<tenant-domain>.onmicrosoft.com'"
Get-MgUserMemberOf -UserId $me.Id
```

Returned a directory role object Id. Resolved the role name/description:

```powershell
Get-MgDirectoryRole -DirectoryRoleId "<role-id>" | Select-Object DisplayName, Description | Format-List
```

Confirmed the account holds **Global Administrator** — expected, since this account is the tenant creator's default admin account.

## Key Takeaways

- Password resets performed via Graph PowerShell never expose the plaintext password back through the API — the only plaintext copy that ever exists is the one typed locally when setting it. This mirrors real-world practice: temporary passwords must be communicated out-of-band at creation/reset time.
- Entra ID enforces permissions through **two independent layers** — Graph API scope consent and directory role (RBAC) assignment. A 403 error can originate from either layer, and troubleshooting requires checking both rather than assuming a re-consent alone will fix it.
- Requesting broad, high-privilege scopes (e.g. `Directory.AccessAsUser.All`) by default is a least-privilege violation. The better practice is to scope each session to only what that specific lab or task requires, and disconnect (`Disconnect-MgGraph`) when finished.
- Cross-verifying object relationships from both directions (e.g. checking group membership via the group, and separately via the user's `MemberOf`) is a useful habit for confirming changes actually took effect, especially in a system with eventually-consistent directory writes.

---

# Troubleshooting Appendix

## Issue 1: Duplicate UserPrincipalName on Second Test User Creation

**Symptom:**

```
New-MgUser_CreateExpanded: Another object with the same value for property userPrincipalName already exists.
Status: 400 (BadRequest)
```

**Diagnosis:** An attempt to create a second test user reused the same `-UserPrincipalName` value from Lab 01's `Test Joiner`, despite giving the new user a different `-DisplayName`. UserPrincipalName must be unique tenant-wide, independent of DisplayName — Entra ID does not treat these two attributes as linked.

**Resolution:** Created the new user with both a distinct DisplayName (`Test Joiner 2`) and a distinct UserPrincipalName (`test.joiner2@<tenant-domain>.onmicrosoft.com`).

## Issue 2: Insufficient Privileges on Password Reset

**Symptom:**

```
Update-MgUser_UpdateExpanded: Insufficient privileges to complete the operation.
Status: 403 (Forbidden)
ErrorCode: Authorization_RequestDenied
```

**Diagnosis:** The active session's granted scopes did not include a permission broad enough to cover password reset operations on another user's account. This operation is more sensitive than standard attribute read/write, and is gated more strictly.

**Resolution:** Reconnected with an expanded scope list including `Directory.AccessAsUser.All`:

```powershell
Disconnect-MgGraph
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All", "Directory.AccessAsUser.All" -TenantId "<tenant-id>" -UseDeviceAuthentication
```

Password reset succeeded after reconnecting with the added scope.

**Lesson:** This surfaced the broader RBAC-vs-scope distinction documented in Part 5 above — a good reminder that Graph 403 errors require checking both the token's consented scopes and the account's assigned directory role, not just one or the other.

## Issue 3: Stale Variables After PowerShell Session Refresh

**Symptom:**

```
New-MgGroupMember: Cannot bind argument to parameter 'GroupId' because it is an empty string.
```

**Diagnosis:** PowerShell was restarted between lab steps, which cleared all session variables (`$newUser2`, `$marketingGroup`, etc.) held in memory. The underlying Entra ID objects were untouched, but the local variable references pointing to them no longer existed.

**Resolution:** Re-fetched the needed objects by a stable identifier (DisplayName filter) to rebuild the variables:

```powershell
$marketingGroup = Get-MgGroup -Filter "displayName eq 'Test Marketing Team'"
$newUser2 = Get-MgUser -Filter "displayName eq 'Test Joiner 2'"
```

**Lesson:** Reinforces that PowerShell variables are session-scoped and disposable — they are a convenience layer over the top of persistent Entra ID object IDs, not a source of truth. Any object needed across sessions should be re-queried by a stable, known identifier (UPN, DisplayName, or Id) rather than assumed to still be held in a variable.
