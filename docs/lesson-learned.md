# Lessons Learned

## Technical

**Autopilot changes where the work happens.** In my AD lab, the work happened before the user ever saw the machine: build a golden image, capture it, deploy it, join the domain. Here the device ships with stock Windows, and everything happens in the cloud after the user signs in. There's no image to maintain or patch.

**Assignment target matters as much as the policy.** A correct compliance policy still failed because it was aimed only at users. Intune also evaluates as the device (System account), so device-level policies need a device group. I now assign compliance to both by default.

**Silent BitLocker has prerequisites.** It needs a TPM, Secure Boot, a policy that allows standard-user encryption, a place to store the recovery key, and no bootable media attached. If any one is missing, it fails quietly, and the only place that says why is the BitLocker Management event log.

**A fresh OS is missing things you take for granted.** At the setup screen, there was no PowerShell Gallery registered, no Graph module, and a sign-in method (WAM) that didn't work yet. Each failure looked different but came from the same root: an unfinished environment.

**Dynamic groups are powerful but strict.** One misplaced quote made the rule invalid. Once it was right, every registered device joined the group automatically, and every profile and policy assigned to that group followed.

## Troubleshooting process

- **Read the message, not just the color.** The DNS failure was buried under a long red stack trace. The useful line was "the remote name could not be resolved."
- **Warnings are clues.** The yellow WAM warning predicted the crash before it happened.
- **Narrow down one layer at a time:** device status → policy → failing rule → the setting behind it → the event log.
- **Verify state before retrying.** The ISO "eject failed" error was misleading; the media was already removed.
- **Don't undo your own fix.** Running a setting with `$true` and then `$false` cancelled the first command.

## Process and documentation

- **Screenshot as you go,** especially failures. The error screenshots ended up being the most useful part of this repo.
- **Separate what the user sees from what IT sees.** In the tickets, the description is the user's report, the internal note holds the investigation, and the customer reply is short and free of jargon.
- **Plan for waiting.** Profile assignment and policy sync each took 15–30 minutes. That time is good for writing documentation.


## What I'd do differently

- Remove the install ISO right after Windows finishes installing.
- Save each policy and confirm it appears in the list before moving on. The missing BitLocker policy cost the most time.
- Assign compliance policies to a device group from the start.
- Run `Get-PSRepository` before installing anything on a fresh machine.


