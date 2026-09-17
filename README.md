# JumpCloud Windows Device Enrollment

A generic, self-service Windows enrollment workflow for **any JumpCloud
tenant**: a polished WPF window asks the employee for their JumpCloud
email and password, verifies the credentials against JumpCloud Cloud
LDAP (LDAPS, no plaintext ever), and only then binds the device to the
user via the JumpCloud API and sets them as primary user.

Deployment is **one command + one zip** — and every tenant-specific
value lives in the command, so you never need to modify the package.

| File | Purpose |
|---|---|
| `MDM-Command.ps1` | The only file you edit. Paste into a JumpCloud PowerShell command. |
| `JumpCloudEnrollment.zip` | The generic enrollment app (UI + logic). Downloaded and SHA-256-verified by the command. Do not edit. |
| `SHA256.txt` | The zip's SHA-256 for manual verification. |

## Setup (5 minutes)

1. Open `MDM-Command.ps1` and edit the marked block — this is the ONLY
   place with tenant values:

   ```powershell
   # ================= TENANT SETTINGS - EDIT THESE =================
   $OrgId          = 'YOUR_JUMPCLOUD_ORG_ID'   # required (console: Settings)
   $Region         = 'US'                      # US or EU tenant
   $CompanyName    = 'Your Organization'
   $SupportContact = 'your IT administrator'
   $AccentColor    = '#0E8A5F'                 # brand color (hex)
   $LogoPath       = ''                        # optional logo PNG on device
   $LdapServer     = 'ldap.jumpcloud.com'      # rarely changed
   $LdapPort       = 636                       # 636 = LDAPS, 389 = StartTLS
   $KeepLogs       = $false                    # $true keeps logs (debugging)
   # ================================================================
   ```

   Your Org ID is in the JumpCloud console under **Settings →
   Organization ID**. If `$OrgId` is left unset, the command refuses to
   run and says so.

2. In the JumpCloud console: **Commands → + New Command** → Windows
   PowerShell, Run As **root**, timeout **3900** seconds (the default
   120 s kills the interactive window).
3. Paste the edited `MDM-Command.ps1`. Keep the two bare
   `{{Apikey}}` / `{{device.id}}` assignments exactly as they are —
   JumpCloud substitutes them at dispatch, quotes included.
4. No attachment needed: `$ZipUrl` already points at this repository's
   `JumpCloudEnrollment.zip`, and the command refuses any zip whose
   SHA-256 does not match the pinned `$ZipSha256`. To deploy the zip as
   a command attachment instead, set `$ZipUrl = ''` — JumpCloud drops
   attachments in `C:\Windows\Temp\`, where the command finds them. The
   attached zip is hash-checked the same way.
5. Save, target your device group, and run.

## When the window appears

Every run prompts the signed-in user. Nothing is recorded on the device,
so the prompt appears again after "Remind Me Later" and after a
completed enrollment. The only run that skips the prompt is one where
nobody is signed in; the command result then reads
`Status: Deferred | Issue: No interactive user`.

If you schedule the command to repeat, remove enrolled devices from the
target group, or they are prompted again on the next run.

## Requirements

- Users must be **enabled for LDAP** in JumpCloud (they authenticate as
  themselves over LDAPS — no service account or shared secret is ever
  placed on endpoints).
- Windows 10/11, 64-bit, Windows PowerShell 5.1 (in-box).
- Devices must not be domain-joined, Azure-AD-joined, or using a
  Microsoft account (the app detects these and shows a friendly
  "contact IT" screen instead).
- Internet access to `ldap.jumpcloud.com:636`, the JumpCloud API,
  `raw.githubusercontent.com` (zip download), and PSGallery (first run
  installs the RunAsUser module).

## What the user sees

Welcome → credential entry (masked password, show/hide, validation) →
progress → success screen with clear next-sign-in steps and
*Sign Out Now / I'll Sign Out Later* — plus friendly distinct screens
for wrong credentials, service unreachable, binding failure (with a
support reference code), and "remind me later".

## Security properties

- The user's password never leaves the UI process — never written to
  disk, registry, logs, IPC, or the SYSTEM process; used once for the
  TLS-protected LDAP bind, then disposed.
- Plaintext LDAP is refused in code, regardless of configuration.
- The command copies the zip into a private folder that only SYSTEM and
  Administrators can open, checks that copy's SHA-256 against the hash
  pinned in the command, and runs exactly the bytes it checked — in both
  download and attachment mode.
- The working files in `C:\ProgramData\JumpCloudEnrollment` are locked:
  every run wipes the folder and re-creates it so that standard users can
  read but never write or plant files there. If the folder can't be
  proven to belong to SYSTEM/Administrators, the run stops.
- The JumpCloud API key is only ever held in memory by the SYSTEM process.
- Unknown email and wrong password produce the identical message (no
  account enumeration).

## What's left on the device

Nothing. When the run ends, whatever the outcome, the command deletes
its private folder, the attached zip, `C:\ProgramData\JumpCloudEnrollment`
and the user's `ui.log`. The only thing kept is the RunAsUser PowerShell
module, which is installed once from PSGallery.

For debugging, set `$KeepLogs = $true`: the SYSTEM log
(`C:\ProgramData\JumpCloudEnrollment\Logs\enrollment.log`, readable by
Administrators only) and `%LOCALAPPDATA%\JumpCloudEnrollment\ui.log` are
then kept. Either way, the one-line `Status:` summary appears in the
command results, and failures are noted in the device's description
field in your JumpCloud console.

## Updating

Always take `MDM-Command.ps1` and `JumpCloudEnrollment.zip` from the
same commit: the pinned hash means an old command deliberately refuses
a newer zip. Re-apply your tenant settings to the new command,
and re-paste it in the console (and replace the attached zip if you use
attachment mode).

## Changelog

- **2026-09-17** — Locked working files and self-destruct. The package
  is hash-checked in a private folder in attachment mode too; the staging
  folder is locked so standard users can't plant or swap files; and
  everything the command put on the device is deleted when it ends. New
  `$KeepLogs` setting keeps the logs for debugging.
- **2026-09-17** — No more registry state. The command no longer skips
  already-enrolled or recently deferred devices; it prompts on every run
  with a signed-in user. "Remind Me Later" closes the window until the
  next run, and the `$DeferMinutes` setting is gone. Devices that ran an
  earlier version may still have an `HKLM\SOFTWARE\JumpCloudEnrollment`
  key; nothing reads it, and it is safe to delete.
- **2026-09-17** — Username alignment now compares the local Windows
  account against the JumpCloud user's System Username when they have
  one, and only falls back to their Username when it is not set. Local
  account renames target the same preferred name.
- Startup no longer stalls 20–30 s on RDP/VDI sessions.
