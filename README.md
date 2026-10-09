# Fixing `helper_unknown_error: setup refresh had errors` on Windows

> A practical, evidence-based troubleshooting guide for Codex Desktop users who cannot start PowerShell, Command Prompt, Git, or another local process.

## The symptom

Codex understands the task, but the first local command fails before PowerShell or another process starts:

```text
Failed to create unified exec process:
helper_unknown_error: setup refresh had errors
```

This message is generic. It does not, by itself, prove that a repository, shell, Git installation, or project file is broken.

## What was tested

| Test | Result | What it showed |
| --- | --- | --- |
| Restart Codex | Failed | A simple restart did not rebuild the failing setup state. |
| Restart Windows | Failed in the reported case | The problem can survive a full reboot. |
| Update Codex | Failed in the reported case | It was not necessarily fixed by the latest installed app version. |
| Run a trivial command | Failed before execution | The failure occurred before the project command ran. |
| Try PowerShell, CMD, and Git | Failed at the same stage | The issue was in the command runner/sandbox layer, not one shell. |
| Run the same check with elevated execution | Worked | The machine could execute commands; the normal sandbox setup path was the failing path. |
| Check the workspace root owner | Normal Windows user | The root folder was not obviously owned by a foreign account. |
| Check for project `.git` and `.codex` folders | Not present in the affected project | The common project-ownership fix did not apply to this project. |
| Open the same files through the normal runner after recovery | Worked | The runtime recovered without changing the project files. |

## What did not fix it

These actions may be reasonable first checks, but they were not sufficient in the affected case:

- Restarting Codex alone.
- Restarting Windows.
- Updating Codex.
- Reopening the project.
- Repeating the same command with a different prompt.
- Assuming the repository or PowerShell was broken.
- Deleting or rebuilding runtime caches without first identifying the failing component.

## What might actually be causing it

The community reports show that this top-level error can have more than one underlying cause. The most plausible causes are:

1. **A sandbox setup or ACL failure.** Codex may be unable to apply the access rules it needs to a workspace or runtime directory. Some users found that `.git` or `.codex` was owned by `CodexSandboxOffline` rather than their normal Windows account.
2. **A locked runtime file.** A running Node/CUA process, antivirus scan, or another process may hold a bundled runtime file open while Codex tries to validate or refresh it.
3. **Path-specific workspace state.** One report recovered by opening the same physical repository through a new Windows junction path, suggesting stale setup state associated with the original absolute path.
4. **The elevated sandbox path itself.** Another report recovered by switching the Windows sandbox from `elevated` to `unelevated`. This is a workaround, not proof that the underlying problem is fixed.
5. **Launching tools as Administrator.** Running Codex or an editor elevated can create a different ownership/permission history and make later sandbox setup more difficult.

The generic message is therefore best treated as a **sandbox/runtime initialization failure**, not as a diagnosis of one specific root cause.

## Recommended recovery order

### 1. Close elevated tools

Close Codex and any editor that was launched as Administrator. In Task Manager, end leftover `ChatGPT.exe` processes if Codex did not fully close. Reopen Codex normally and test a harmless command.

### 2. Inspect ownership before changing permissions

From an administrator PowerShell, inspect the exact folder named by the sandbox log:

```powershell
(Get-Acl ".").Owner
(Get-Acl ".git").Owner
(Get-Acl ".codex").Owner
```

Only inspect paths that actually exist. If the log identifies `.git` or `.codex` and its owner is `CodexSandboxOffline` instead of your normal Windows account, restore ownership of that specific folder only:

```powershell
icacls ".git" /setowner "YOUR-COMPUTER-NAME\\YOUR-USER-NAME"
```

Do not blindly reset all permissions, delete the repository, or disable Windows security. First identify the exact path and error code.

### 3. Try the temporary sandbox workaround

Back up `%USERPROFILE%\\.codex\\config.toml`, then change:

```toml
[windows]
sandbox = "elevated"
```

to:

```toml
[windows]
sandbox = "unelevated"
```

Restart Codex normally and test again. `unelevated` changes the isolation model, so treat this as a temporary diagnostic/workaround and restore the previous setting if it does not help.

### 4. Test a fresh path

If the failure is isolated to one workspace, test an empty workspace or a new path. A Windows directory junction can be used as a reversible path-level workaround, but it should not replace the repository or alter Git history.

### 5. Capture the underlying log

The useful diagnosis is usually the nested error underneath the generic message—for example `SetNamedSecurityInfoW failed: 5` for access denied or an OS error indicating that a runtime file is locked. Do not publish logs containing usernames, tokens, or local paths without redacting them.

## What not to do first

- Do not delete `%USERPROFILE%\\.codex` or the runtime cache as a first response.
- Do not run every editor and Codex as Administrator.
- Do not reset all ACLs on the repository.
- Do not disable Defender or other security tools just because the error is vague.
- Do not assume a failed local command means Git, PowerShell, or the repository is corrupt.

## Bottom line

When elevated execution works but the normal runner fails, the strongest working hypothesis is a **Codex Windows sandbox setup problem**—often permissions, ownership, file locking, or stale path-specific state. The safest sequence is: close elevated apps, inspect the exact failing path, try the documented `unelevated` workaround, and only then consider a path alias or targeted ownership repair.

## Sources

- [OpenAI Developer Community discussion](https://community.openai.com/t/codex-desktop-command-runner-completely-broken-helper-unknown-error-setup-refresh-had-errors/1391898)

This guide is a community troubleshooting summary, not an official OpenAI support procedure. Commands that change ownership or sandbox behavior should be used carefully.
