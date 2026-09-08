# JumpCloud Enrollment — deployment package

Minimal deployment for a JumpCloud Windows device-enrollment workflow:
**one command, one zip.**

| File | Purpose |
|---|---|
| `MDM-Command.ps1` | Paste into a JumpCloud PowerShell command (**Run As: root**, timeout **≥ 3900 s**). No attachment needed. |
| `JumpCloudEnrollment.zip` | The enrollment app. The command downloads it from this repository and refuses to run it unless its SHA-256 matches the hash pinned inside the command. |
| `SHA256.txt` | The zip's SHA-256, for manual verification. |

## Deploy

1. In the JumpCloud console: **Commands → + New Command** → Windows
   PowerShell, Run As **root**, timeout **3900**.
2. Paste the entire contents of `MDM-Command.ps1`. Leave the two bare
   template-variable assignments exactly as they are — JumpCloud
   substitutes them (quotes included) at dispatch.
3. Save, target the device group, run (or schedule to repeat — deferred
   and completed devices exit quietly in under a second).

## Update

Never replace `JumpCloudEnrollment.zip` alone: the pinned hash in the
old command will (by design) refuse the new zip. Publish the freshly
built `MDM-Command.ps1` + `JumpCloudEnrollment.zip` + `SHA256.txt`
together, then re-paste the command in the JumpCloud console.

Source, build tooling, tests, and full documentation live in the
project's source repository.
