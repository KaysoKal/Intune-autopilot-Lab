# Lab Notes

Plain-language notes I took while building this lab. They cover the commands I used and the concepts behind each step.

---

## Hyper-V and the VM

**Checking Hyper-V**
```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```
`Get-WindowsOptionalFeature` checks whether a Windows feature is turned on. Hyper-V is the built-in Windows tool for virtual machines. It must say **Enabled**.

**Virtual hard disk vs. ISO**
- **Virtual hard disk (.vhdx):** the VM's blank storage drive, saved as a single file on my real PC.
- **ISO:** the Windows installer, like a virtual install disc.
- On first boot, the VM runs the installer from the ISO and puts Windows on the virtual hard disk. Same idea as a technician building a PC with an empty SSD and a Windows USB stick.

**Golden image vs. Autopilot**
- **Golden image:** IT builds and maintains a customized Windows image, then copies it to every PC (traditional).
- **Autopilot:** start from the stock Windows the PC ships with and let Intune apply settings and apps from the cloud (modern). No image to maintain.
- I used a fresh ISO instead of my AD lab's golden image, so this lab shows the modern method cleanly.

**TPM before first boot.** Windows 11, BitLocker, and the Autopilot hardware hash all depend on the TPM. Turn it on in VM Settings → Security before starting the VM.

**Checkpoints.** A checkpoint is a saved snapshot. **Apply** rolls the VM back and loses everything since. **Delete** removes the snapshot without changing the VM. Never revert a device enrolled in Intune or Entra, because the cloud still has the newer state.

---

## Entra ID and Intune setup

**Automatic enrollment (MDM user scope: All).** Tells Intune to manage any Windows device my users sign in to.

**Dynamic device group rule**
```
(device.devicePhysicalIDs -any (_ -startsWith "[ZTDId]"))
```
`[ZTDId]` is a tag Autopilot adds to every device registered with a hardware hash (ZTD = Zero Touch Deployment). The rule means "add any device with that tag," so registered devices join automatically.

**Device name template.** `LAB-%RAND:4%` = "LAB-" plus 4 random digits (example: LAB-6622). Names must be 15 characters or fewer. `%SERIAL%` is the other common option. Companies use patterns like `NYC-LT-%RAND:5%` for a New York laptop.

**Enrollment Status Page (ESP).** The progress screen during first sign-in. It shows device setup and account setup, and can block the desktop until required apps and policies are installed.

**Compliance policy.** A checklist of rules (password, encryption, OS version). Intune marks the device Compliant or Not compliant. With Conditional Access, non-compliant devices can be blocked from email and company apps. `10.0.22000` = the first Windows 11 release, so any Windows 11 build passes.

**BitLocker policy.** Encrypts the drive silently. "Allow Warning For Other Disk Encryption: Disabled" plus "Allow Standard User Encryption: Enabled" lets it encrypt without a popup and without the user being an admin. The recovery key is backed up to Entra ID, so help desk can find it in Intune if a user gets locked out. The "AD DS" labels are old names; on Entra-joined devices the key goes to Entra ID.

**Update rings.** Control update timing.
- Quality updates = monthly security patches. Feature updates = the yearly Windows versions.
- Deferral = days to wait after Microsoft releases an update.
- Common setup: a pilot ring (0 days) for IT, then a broad ring (7–14 days) for everyone else.

**Required vs. Available apps.** *Required* installs automatically. *Available* shows up in Company Portal for the user to install if they want.

---

## The hardware hash: registering the device

### The big idea

- A **hardware hash** is the device's **fingerprint**: a long code made from its parts (serial number, TPM chip, and so on).
- I upload that fingerprint to Intune. That tells Microsoft, **"This device belongs to my company."**
- Later, when the device turns on and connects to the internet, Microsoft recognizes the fingerprint and sends it my company's setup instructions (the Autopilot profile).

**Think of it like this:** it's like adding a new employee's badge number to the system before their first day. When they show up and tap the badge, the door already knows who they are.

**In the real world:** the company usually buys laptops from Dell, HP, or Lenovo, and the vendor uploads the hash for them. In a lab, I do it myself with a script.

### Where I did it

1. Let Windows install and stop at the **"Let's set things up for your work or school"** screen. **Don't sign in.**
2. Press **Shift + F10**. A black Command Prompt window opens.
3. Type `powershell` and press **Enter**. The line should now start with **PS**. That means I'm in PowerShell.

### The steps, in order

**Step 1: Check the security chip**
```powershell
Get-Tpm
```
The TPM is a security chip. Look for `TpmPresent : True`.
⚠️ Don't share screenshots that show the **OwnerAuth** line. It's a secret.

**Step 2: Allow scripts to run**
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```
Windows blocks scripts by default. This allows them, but only in this one window. It resets when I close it.

**Step 3: Get the download tool**
```powershell
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
```
NuGet is like the delivery truck PowerShell uses to download tools. A fresh Windows install doesn't have it yet.

**Step 4: Make sure PowerShell knows where to download from**
```powershell
Get-PSRepository
```
If it says **"Unable to find module repositories,"** run:
```powershell
Register-PSRepository -Default
```
The PowerShell Gallery (PSGallery) is like an app store for PowerShell tools. This command adds it back.

**Step 5: Install the Microsoft sign-in tool**
```powershell
Install-Module Microsoft.Graph.Authentication -Force -Scope AllUsers
```
This lets PowerShell sign in to my Microsoft 365 tenant.

**Step 6: Turn off the sign-in popup that crashes**
```powershell
Set-MgGraphOption -DisableLoginByWAM $true
```
WAM is Windows' built-in sign-in popup, and it doesn't work yet at the setup screen. This switches to a normal browser sign-in.
Only run this once. `$true` turns it off; running it again with `$false` turns WAM back on.

**Step 7: Download the Autopilot script**
```powershell
Install-Script -Name Get-WindowsAutopilotInfo -Force
```
Type **Y** if it asks a question.

**Step 8: Read the fingerprint and upload it**
```powershell
Get-WindowsAutopilotInfo -Online
```

**Step 9: Sign in**
- Choose **Work or school account**, not Microsoft account.
- Sign in with my **admin** account.
- If asked, choose **"sign in to this app only."**
- Accept the permissions screen. It's Microsoft's own tool (verified publisher), and it needs permission to add the device to Intune.

**Step 10: Wait for the success messages**
- `1 devices imported successfully`
- `All devices synced`

### After the upload: three statuses

In **Intune → Devices → Windows → Enrollment → Windows Autopilot → Devices**:

| Status | What it means |
|---|---|
| **Imported** | Intune got the fingerprint |
| **Synced** | Autopilot knows the device |
| **Assigned** | The device has its setup instructions. **Wait for this before restarting.** |

Then restart from the same PowerShell window:
```powershell
shutdown /r /t 0
```
`/r` = restart, and `/t 0` = do it now.

### If something goes wrong

| What I see | What it means | What to do |
|---|---|---|
| `Get-Tpm` "is not recognized" | I'm in Command Prompt, not PowerShell | Type `powershell` first |
| PowerShell closes during sign-in | The WAM sign-in popup crashed | Do Step 6, then try again |
| `Set-MgGraphOption` "is not recognized" | The sign-in tool isn't installed | Do Step 5 first |
| "No match was found" | PowerShell has no app store to download from | Do Step 4 |
| `Get-WindowsAutopilotInfo` "is not recognized" | PowerShell can't find the script | Check spelling, or run it by its full path (below) |
| "The remote name could not be resolved" | DNS problem: the VM can't look up Microsoft's address | Run the DNS checks (below) |

**Run the script by its full path:**
```powershell
& "C:\Program Files\WindowsPowerShell\Scripts\Get-WindowsAutopilotInfo.ps1" -Online
```
The `&` means "run what's in these quotes." Watch the spelling: it's **Windows**PowerShell, with an **s**.

**DNS checks:**
```powershell
Resolve-DnsName graph.microsoft.com
```
If it shows IP addresses, DNS works, so run Step 8 again. If it shows an error, run:
```powershell
ipconfig /flushdns
```
That clears old lookups. Then try again.

**Tip:** to paste commands into the VM, use the VM window's **Clipboard → Type clipboard text**. That way I don't have to retype anything.

### Words to know

| Word | Plain meaning |
|---|---|
| Hardware hash | The device's fingerprint |
| OOBE | The first Windows setup screens ("out-of-box experience") |
| TPM | A security chip that protects keys and passwords |
| Module | A toolkit of PowerShell commands |
| Script | A single file of commands that runs like a program |
| PSGallery | The online app store for PowerShell tools |
| Microsoft Graph | The connection scripts use to talk to Microsoft 365 and Intune |
| WAM | Windows' built-in sign-in popup |
| DNS | Turns names like graph.microsoft.com into IP addresses |

---

## Verifying and fixing the device

```powershell
hostname
```
Shows the computer name. `LAB-6622` proves the naming template worked.

**Elevating as a standard user.** On Entra-joined devices, Global Admins are automatically local admins, so I can run PowerShell as administrator with my admin account while signed in as a standard user.

```powershell
manage-bde -status C:
```
Shows BitLocker's state on C: **Conversion Status** (encrypted?), **Protection Status** (actively protected?), and **Key Protectors** (TPM, recovery password).

**BitLocker states**
- "Waiting for activation" = the drive is encrypted, but the key isn't protected yet (no TPM protector, no recovery key backup).
- Silent encryption won't finish while bootable media (ISO, USB) is attached, because that media could be used to bypass the TPM boot check.

```powershell
Get-WinEvent -LogName "Microsoft-Windows-BitLocker/BitLocker Management" -MaxEvents 10 |
    Format-List TimeCreated, Id, Message
```
Reads the BitLocker Management log. Event IDs to know:

| Event | Meaning |
|---|---|
| 853 | Silent encryption failed (here: bootable media detected) |
| 845 | Recovery key backed up to Entra ID |
| 775 | Key protector created |
| 892 | Key sealed to the TPM |
| 774 | Protection resumed |

**Reading compliance.** Open the device's compliance report, then open the policy to see which rule failed. Narrow it down one layer at a time instead of guessing.

**Error 65001 on the Default Device Compliance Policy.** Usually means the real compliance policy is assigned only to users. Intune also checks as the System account (device context), which sees no policy. Assign compliance to both a user group and a device group.

**Sync.** Devices check in on a schedule (roughly every 8 hours once settled). Sync forces an immediate check-in. Help desk does it from Intune; users can do it from **Settings → Accounts → Access work or school → Info → Sync**.

**Company Portal → Devices → Check status.** Forces an immediate compliance re-check. A common help desk instruction.

**Remote Restart.** Reboots a user's device from Intune without touching it. **Device action status** shows whether the command reached the device and completed.
