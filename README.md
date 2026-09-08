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
   $DeferMinutes   = 120                       # Remind Me Later snooze
   $LdapServer     = 'ldap.jumpcloud.com'      # rarely changed
   $LdapPort       = 636                       # 636 = LDAPS, 389 = StartTLS
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
4. Save, target your device group, run — or schedule it to repeat:
   deferred and already-enrolled devices exit quietly in under a second,
   which is what makes "Remind Me Later" re-prompt later.

## Requirements

- Users must be **enabled for LDAP** in JumpCloud (they authenticate as
  themselves over LDAPS — no service account or shared secret is ever
  placed on endpoints).
- Windows 10/11, 64-bit, Windows PowerShell 5.1 (in-box).
- Devices must not be domain-joined, Azure-AD-joined, or using a
  Microsoft account (the app detects these and shows a friendly
  "contact IT" screen instead).
- Internet access to `ldap.jumpcloud.com:636`, the JumpCloud API, this
  repository (zip download), and PSGallery (first run installs the
  RunAsUser module).

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
- The command only runs a zip whose SHA-256 matches the hash pinned
  inside the command, so a swapped download can't execute as SYSTEM.
- Unknown email and wrong password produce the identical message (no
  account enumeration).
- Diagnostics go to `C:\ProgramData\JumpCloudEnrollment\Logs`
  (SYSTEM/Administrators only) and, on failure, into the device's
  description field in your JumpCloud console.

## Updating

Always take `MDM-Command.ps1` and `JumpCloudEnrollment.zip` from the
same commit: the pinned hash means an old command deliberately refuses
a newer zip. Re-apply your tenant settings to the new command and
re-paste it in the console.
