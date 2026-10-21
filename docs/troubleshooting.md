# Troubleshooting Log

13 issues across three areas. For each one: what happened, why, how I fixed it, and how I'd explain it in one or two sentences.

| Area | Issues | The short version |
|---|---|---|
| Hardware hash upload | #2–#8 | A fresh Windows install was missing the tools and connections the upload script needed. I fixed them one layer at a time. |
| BitLocker | #9–#11 | The device was Not compliant because BitLocker never finished turning on. The event log showed the install ISO was still attached. |
| Compliance | #12–#13 | The compliance policy only targeted users, so the device-level check failed. I assigned it to the device group too. |

**My method:** read the actual error message → narrow it down one layer at a time → check logs before guessing → confirm the real state before retrying.

---

## 1. Dynamic group rule rejected as invalid
- **Problem:** Intune rejected my `Autopilot-Devices` group rule as invalid.
- **Cause:** Copying and pasting added extra quote marks.
- **Fix:** Retyped it exactly: `(device.devicePhysicalIDs -any (_ -startsWith "[ZTDId]"))`
- **Say it:** "A paste added quote marks to my dynamic group rule. Once I typed it exactly, every Autopilot device joined the group automatically."
- **Proof:** [04](../screenshots/04-dynamic-group-rule-invalid.png)

## 2. Get-Tpm not recognized
- **Problem:** `Get-Tpm` said "not recognized" at the Windows setup screen.
- **Cause:** Shift+F10 opens Command Prompt, not PowerShell.
- **Fix:** Typed `powershell` first, then `Get-Tpm` returned `TpmPresent : True`.
- **Say it:** "I was in Command Prompt, not PowerShell. Now I always check that the prompt starts with PS."
- **Proof:** [10](../screenshots/10-tpm-check.png)

## 3. PowerShell closed during sign-in (WAM)
- **Problem:** The upload script opened a sign-in, then PowerShell closed. Nothing uploaded.
- **Cause:** The Microsoft Graph module signs in through WAM (Windows' built-in sign-in) by default, and WAM doesn't work reliably before Windows setup is finished.
- **Fix:** Ran `Set-MgGraphOption -DisableLoginByWAM $true` to switch to a normal browser sign-in.
- **Say it:** "A yellow warning about WAM showed up right before the crash. Turning WAM off fixed the sign-in."
- **Proof:** [12](../screenshots/12-hash-wam-crash.png)

## 4. Script not recognized and WAM fix undone
- **Problem:** `Get-WindowsAutopilotInfo` said "not recognized," and WAM was back on.
- **Cause:** The scripts folder wasn't on PowerShell's search path (PATH). I'd also run the WAM setting with `$true` and then `$false`, which undid it.
- **Fix:** Ran the script by its full path with `&`, and set WAM only once.
- **Say it:** "'Not recognized' means PowerShell can't find the command, so I ran it by its full path. I also caught that I'd undone my own fix."
- **Proof:** [13](../screenshots/13-hash-script-not-found.png)

## 5. Set-MgGraphOption not recognized
- **Problem:** After rebuilding the VM, `Set-MgGraphOption` didn't exist.
- **Cause:** That command comes from the `Microsoft.Graph.Authentication` module, which wasn't installed on the fresh OS.
- **Fix:** Installed the module first (which led to #6).
- **Say it:** "A command only exists if its module is installed. On a fresh machine, install order matters."
- **Proof:** [14](../screenshots/14-graph-module-missing.png)

## 6. Install-Module: no match was found
- **Problem:** `Install-Module` said "No match was found."
- **Cause:** `Get-PSRepository` showed no repositories. The PowerShell Gallery wasn't registered, so PowerShell had nowhere to download from. The internet was fine, because NuGet had just downloaded.
- **Fix:** `Register-PSRepository -Default`, then installed the module and ran the upload. Sign-in worked.
- **Say it:** "It looked like a typo, but there was no download source registered. NuGet downloading proved the network was fine, so I knew to check the repository."
- **Proof:** [15](../screenshots/15-psgallery-no-match.png), [16](../screenshots/16-psgallery-registered.png)

## 7. DNS could not resolve graph.microsoft.com
- **Problem:** Sign-in worked and the script read the serial number, then failed.
- **Cause:** The error said "the remote name could not be resolved." That's a temporary DNS failure on the Hyper-V Default Switch.
- **Fix:** `Resolve-DnsName graph.microsoft.com` confirmed DNS was working again, and I re-ran the upload. (Backup plan: `ipconfig /flushdns` and set DNS to 8.8.8.8.)
- **Say it:** "The stack trace was long, but the real line said the name couldn't be resolved. That's DNS, not sign-in."
- **Proof:** [17](../screenshots/17-hash-dns-failure.png)

## 8. Script path typo
- **Problem:** The full-path command said "not recognized" again.
- **Cause:** I typed `WindowPowershell` instead of `WindowsPowerShell`.
- **Fix:** Used the short command `Get-WindowsAutopilotInfo -Online`. Result: `1 devices imported successfully`.
- **Say it:** "Simple typo. Now I paste long commands into the VM with Hyper-V's Type clipboard text."
- **Proof:** [18](../screenshots/18-hash-import-success.png)

---

## 9. Encryption rule failed: no BitLocker policy
- **Problem:** After Autopilot finished, the device was **Not compliant**. Password and OS version passed, but encryption failed (`2016281112`). BitLocker said "waiting for activation."
- **Cause:** The BitLocker policy had never saved. Nothing told the device to finish turning BitLocker on.
- **Fix:** Created `Lab_BitLocker` and assigned it to `Autopilot-Devices`. It applied, but BitLocker still didn't activate (#10).
- **Say it:** "I narrowed it down from the device to the policy to the one failing rule, which was encryption. Then I found the BitLocker policy didn't exist."
- **Proof:** [24](../screenshots/24-device-not-compliant.png), [26](../screenshots/26-encryption-rule-error.png), [27](../screenshots/27-bitlocker-waiting-for-activation.png)

## 10. BitLocker blocked by attached ISO (Event 853)
- **Problem:** The policy applied, but `manage-bde -status C:` showed **Protection Off** with **no key protectors**.
- **Cause:** The BitLocker event log showed **Event 853: bootable media detected**. The Windows install ISO was still attached. BitLocker won't lock the key to the TPM while bootable media is present, because that media could be used to get around the TPM's security check.
- **Fix:** Removed the ISO (DVD Drive → Media: None), restarted, and synced. The log then showed success events 845, 775, 892, and 774, and `manage-bde` showed **Protection On** with TPM and recovery password.
- **Say it:** "manage-bde showed protection off, so I checked the BitLocker event log. Event 853 said bootable media was attached: the install ISO was still in the virtual DVD drive. I removed it and restarted, and BitLocker sealed the key to the TPM and backed up the recovery key to Entra ID. It's the same issue as a tech leaving an install USB in a new laptop."
- **Proof:** [30](../screenshots/30-manage-bde-event-853.png), [32](../screenshots/32-bitlocker-success-events.png), [33](../screenshots/33-manage-bde-protection-on.png) · Ticket: [HELP-3](../tickets/HELP-3-bitlocker-blocked-by-iso.md)

## 11. Access denied when ejecting the ISO
- **Problem:** Ejecting the ISO from the VM window gave "access was denied."
- **Cause:** Hyper-V can block media changes while a VM is running, but the eject had actually worked.
- **Fix:** Checked VM Settings with the VM off: DVD Drive already showed **Media: None**.
- **Say it:** "The error didn't match what happened, so I checked the actual setting before retrying."
- **Proof:** [31](../screenshots/31-iso-removed-dvd-none.png)

---

## 12. Default policy error 65001
- **Problem:** With BitLocker fixed, the device still wasn't compliant. The Default policy showed **"Has a compliance policy assigned: Error 65001."**
- **Cause:** Intune checks compliance twice: as the signed-in user and as the device itself (System account). My policy was assigned only to `Lab-Users`, so the device check found no policy.
- **Fix:** Added `Autopilot-Devices` to the policy's assignments and kept `Lab-Users`. Then I sent a remote Sync and ran Check status in Company Portal.
- **Say it:** "Intune checks compliance as the user and as the device. My policy only targeted users, so I assigned it to the device group too. Now I assign compliance to both by default."
- **Proof:** [34](../screenshots/34-default-policy-65001.png), [35](../screenshots/35-device-compliant.png) · Ticket: [HELP-4](../tickets/HELP-4-compliance-policy-assignment.md)

## 13. Sync status: Policies partial success
- **Problem:** The device showed **Compliant**, but one report still said 1 Not compliant, and the sync showed "partial success."
- **Cause:** Some reports from before the fixes hadn't refreshed yet.
- **Fix:** Sent a **remote Restart** from Intune. After it checked back in: **Compliant 2, Succeeded 4, Installed 1**.
- **Say it:** "A Sync updated the main status, but some reports only refreshed after a restart. The remote restart cleared everything."
- **Proof:** [39](../screenshots/39-compliant-after-remote-restart.png)
