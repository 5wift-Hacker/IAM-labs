# Lab 01: Manual Joiner-Mover-Leaver (JML) Operations via Microsoft Graph PowerShell

## Objective

Practice manual Joiner-Mover-Leaver (JML) identity lifecycle operations using the Microsoft Graph PowerShell SDK against a Microsoft Entra ID tenant. The goal was to understand the underlying mechanics that Entra ID Governance's Lifecycle Workflows automate, before working with the fully licensed Governance workflow engine.

## Environment

- **Tenant**: Microsoft Entra ID tenant (Azure account)
- **Tools**: Microsoft Graph PowerShell SDK
- **PowerShell version**: 7.6.3 (latest)
- **Auth method**: Delegated, device code flow
- **Scopes used**: `User.ReadWrite.All`, `Group.ReadWrite.All`, `Directory.ReadWrite.All`

## Part 1: Connecting to Microsoft Graph

Connected using device code authentication after running into an interactive/WAM auth failure (see Troubleshooting section below).

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All" `
    -TenantId "<tenant-id>" `
    -UseDeviceAuthentication
```

Verified the session:

```powershell
Get-MgContext
```

Confirmed `TenantId`, `Scopes`, and `AuthType` populated correctly before proceeding.

## Part 2: Joiner — Provisioning a New User

Created a test user representing a new hire:

```powershell
$PasswordProfile = @{
    Password = "TempP@ssw0rd123!"
    ForceChangePasswordNextSignIn = $true
}

$newUser = New-MgUser -DisplayName "Test Joiner" `
    -UserPrincipalName "test.joiner@<tenant-domain>.onmicrosoft.com" `
    -AccountEnabled `
    -MailNickname "testjoiner" `
    -PasswordProfile $PasswordProfile `
    -Department "Sales"
```

Created two test security groups to simulate department-based access:

```powershell
$salesGroup = New-MgGroup -DisplayName "Test Sales Team" -MailEnabled:$false -MailNickname "testsales" -SecurityEnabled:$true

$marketingGroup = New-MgGroup -DisplayName "Test Marketing Team" -MailEnabled:$false -MailNickname "testmarketing" -SecurityEnabled:$true
```

Added the new user to the Sales group (initial access grant):

```powershell
New-MgGroupMember -GroupId $salesGroup.Id -DirectoryObjectId $newUser.Id
```

**Verification:**

```powershell
Get-MgGroupMember -GroupId $salesGroup.Id
```

Confirmed the user's object ID appeared as a member of Test Sales Team.

## Part 3: Mover — Changing Department and Access

Simulated an internal transfer from Sales to Marketing:

```powershell
# Remove from old group
Remove-MgGroupMemberByRef -GroupId $salesGroup.Id -DirectoryObjectId $newUser.Id

# Add to new group
New-MgGroupMember -GroupId $marketingGroup.Id -DirectoryObjectId $newUser.Id

# Update department attribute to reflect the move
Update-MgUser -UserId $newUser.Id -Department "Marketing"
```

**Verification:**

```powershell
Get-MgGroupMember -GroupId $salesGroup.Id       # Empty — confirms removal
Get-MgGroupMember -GroupId $marketingGroup.Id   # Shows user — confirms addition
Get-MgUser -UserId $newUser.Id -Property DisplayName, Department | Format-List
```

Result: `Department: Marketing` confirmed, group membership correctly reflected the transfer.

## Part 4: Leaver — Offboarding

Simulated an employee departure using a phased approach (disable → strip access → optional delete):

**Step 1 — Disable sign-in:**

```powershell
Update-MgUser -UserId $newUser.Id -AccountEnabled:$false
```

**Step 2 — Remove all group memberships:**

```powershell
Get-MgUserMemberOf -UserId $newUser.Id | ForEach-Object {
    Remove-MgGroupMemberByRef -GroupId $_.Id -DirectoryObjectId $newUser.Id
}
```

**Verification:**

```powershell
Get-MgUser -UserId $newUser.Id -Property DisplayName, AccountEnabled | Format-List
Get-MgUserMemberOf -UserId $newUser.Id
```

Result: `AccountEnabled: False`, no group memberships returned — access fully revoked while account preserved for audit/reference.

**Step 3 — Full deletion (optional, tested separately):**

```powershell
Remove-MgUser -UserId $newUser.Id
```

This moves the account into Entra ID's soft-delete state (30-day recovery window) rather than permanently destroying it immediately.

## Key Takeaways

- Manually performing JML operations makes clear exactly what Entra ID Governance's Lifecycle Workflows automate under the hood: attribute updates, group membership changes, and account enable/disable state, all triggered by conditions like `employeeHireDate` and `employeeLeaveDateTime`.
- Object references in Graph PowerShell are handled via the object's immutable `Id` (GUID), not its display name. Display names are human-readable labels only; they are not accepted as identifiers by cmdlets expecting `-UserId` or `-GroupId`.
- Entra ID's soft-delete behavior for deleted users provides a 30-day recovery window, which proved directly useful during this lab (see Troubleshooting Appendix).

---

# Troubleshooting Appendix

Documenting the issues hit during this lab, since diagnosing and resolving these was arguably the more valuable part of the exercise.

## Issue 1: WAM Authentication Failure

**Symptom:**

```
Azure.Identity.AuthenticationFailedException: InteractiveBrowserCredential authentication failed
MsalServiceException: WAM Error, Error Code: 0, IncorrectConfiguration
```

**Diagnosis:** Windows Web Account Manager (WAM), the native broker used by `Connect-MgGraph` for interactive sign-in, failed to authenticate — this was a local Windows/client configuration issue, not an Entra ID account problem.

**Resolution:** Bypassed WAM entirely by switching to device code authentication:

```powershell
Connect-MgGraph -Scopes "User.Read" -UseDeviceAuthentication
```

After using device code auth successfully, a subsequent attempt at a direct/interactive connection (`Connect-MgGraph` without `-UseDeviceAuthentication`) succeeded — suggesting the initial WAM failure may have been a transient issue rather than a persistent configuration problem.

## Issue 2: Insufficient Privileges on Graph Query

**Symptom:**

```
Get-MgUser_List: Insufficient privileges to complete the operation.
Status: 403 (Forbidden)
ErrorCode: Authorization_RequestDenied
```

**Diagnosis:** The session had connected successfully, but only with the `User.Read` scope (read own profile only) — insufficient for listing all users in the tenant.

**Resolution:** Reconnected with a broader delegated scope:

```powershell
Connect-MgGraph -Scopes "User.Read.All" -TenantId "<tenant-id>" -UseDeviceAuthentication
```

**Lesson:** A successful connection does not guarantee sufficient permissions for a given operation — `Get-MgContext` should be checked to confirm which scopes were actually granted during consent, since Graph PowerShell scopes are least-privilege by design and are not pre-authorized.

## Issue 3: Accidental Permanent-Looking Deletion

**Symptom:** Ran `Remove-MgUser` on the test account as the final optional step of the Leaver process, then later tested recovery of the account.

**Diagnosis:** `Remove-MgUser` does not immediately destroy an Entra ID object — it moves it into a soft-delete state, recoverable for 30 days.

**Resolution:**

    Restore-MgDirectoryDeletedItem -DirectoryObjectId "<object-id>"

Here, `<object-id>` is the deleted user's Object ID (GUID) — in this lab, `$newUser.Id`, since the variable was still live in the same PowerShell session:

    Restore-MgDirectoryDeletedItem -DirectoryObjectId $newUser.Id

**Verification:**

    Get-MgUser -UserId $newUser.Id -Property DisplayName, AccountEnabled | Format-List

Confirmed the user object was restored with all prior attributes intact (including its pre-deletion `AccountEnabled: False` state).

**Lesson:** Understanding Entra ID's soft-delete/recovery behavior is directly applicable to real-world offboarding processes. Accidental deletions during automation or manual admin work are recoverable within the retention window, which is an important safety net to know how to use under pressure — but only if you still have a way to reference the object's ID (see Issue 4).

## Issue 4: Losing the Object Reference After Closing the Session

**Symptom:** Simulated a more realistic scenario by closing the PowerShell session after deleting the test user, then reopening a new session to attempt recovery.

**Diagnosis:** `$newUser` and its `.Id` property only exist in memory for the lifetime of the session. Once PowerShell is closed (or the variable is overwritten), that reference is gone — even though the underlying GUID itself hasn't changed and the object is still recoverable in Entra ID's soft-delete state.

Attempting the same recovery command in the new session failed immediately, since the variable no longer existed:

    Restore-MgDirectoryDeletedItem -DirectoryObjectId $newUser.Id

    The term '$newUser' is either not recognized... (variable is $null)

**Resolution:** Without the original variable, the Object ID has to be looked up from Entra ID's deleted-items list directly, filtering on a known attribute like display name:

    Get-MgDirectoryDeletedItemAsUser -All | Where-Object DisplayName -eq "Test Joiner"

This returns the deleted user object, including its `Id`, which can then be fed into the restore cmdlet:

    $deletedUser = Get-MgDirectoryDeletedItemAsUser -All | Where-Object DisplayName -eq "Test Joiner"
    Restore-MgDirectoryDeletedItem -DirectoryObjectId $deletedUser.Id

**Lesson:** Session-scoped variables are not a durable record of an object's identity — they're a convenience that disappears the moment the session ends. In a real offboarding/recovery scenario, relying on `$newUser.Id` staying available is fragile; the more resilient habit is to log or export critical Object IDs (e.g., to a CSV or ticketing system) at the time of action, rather than assuming they'll still be sitting in memory later. If that habit isn't in place, recovery is still possible, but it takes an extra lookup step and requires knowing at least one other identifying attribute (display name, UPN, etc.) to filter on.
