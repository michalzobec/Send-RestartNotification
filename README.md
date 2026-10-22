# Send Restart Notification

Copyright (c) 2021-2026 Michal Zobec, ZOBEC Consulting. All Rights Reserved.  
Web: [www.michalzobec.cz](https://www.michalzobec.cz)  
Licensed under GNU General Public License 3

Sends an e-mail notification when a Windows server (or a special-purpose Windows workstation) is going down and when it is back online. The notification tells who requested the restart or power off, how long the outage took and whether the previous shutdown was clean or unexpected.

Release notes are in [CHANGELOG.md](CHANGELOG.md).

## Contents

| File | Purpose |
|---|---|
| `Send-RestartNotification.ps1` | Notification script (`-CallAction Shutdown`, `Startup`, `Test`) |
| `Send-RestartNotification.cmd` | Runs the notification script with the given parameters |
| `1Install-RestartNotification.ps1` | Installer: interactive install, reinstall and uninstall, or unattended install with `-ConfigName` |
| `1Install-RestartNotification.cmd` | Runs the installer interactively |
| `img\setup.ico` | Icon of the Programs and Features entry |
| `config\Send-RestartNotification-EXAMPLE.psd1` | Sample configuration (SMTP, company name, colors, texts), template for `Send-RestartNotification-<name>.psd1` |
| `config\img\banner.png` | Banner at the top of the e-mail, 600 x 140 px, decoration only |
| `config\img\logo.png` | Company logo next to the company name below the banner, square, shown at 40 x 40 px |
| `README.md` | This document |
| `NOTES.md` | Background notes: why things are set up the way they are |
| `CHANGELOG.md` | Release notes |
| `LICENSE` | GNU General Public License 3 |

## How it works

| When | Started by | Action | E-mail |
|---|---|---|---|
| Restart or power off is requested | Local Group Policy, PowerShell shutdown script | `-CallAction Shutdown` | Restart requested / Power off requested |
| Operating system has started | Scheduled task `\ZOBEC\Send Restart Notification` | `-CallAction Startup` | Back online, or Unexpected restart |

**Shutdown.** The script reads the latest event 1074 (User32) since the last boot: process, account, reason, comment and shutdown type. A shutdown from the logon screen (winlogon.exe running as SYSTEM) is shown as *Logon screen (no user signed in)*. The script tries to reach the SMTP server several times (see Configuration).

Every notification starts with a summary (Downtime, Action, Initiated by) followed by the details.

Known processes get a short explanation in the details:

| Process | Explanation | Verified |
|---|---|---|
| RuntimeBroker.exe | Action invoked by the user from the Start menu | yes |
| winlogon.exe (SYSTEM) | Logon screen (no user signed in) | yes |
| winlogon.exe (signed-in user) | Windows logon process (Ctrl+Alt+Del screen) | no |
| shutdown.exe | Shutdown command (command line, script or remote tool) | no |
| shutdown.exe, comment contains "Ansible" | Ansible (win_reboot module) | no |
| MoUsoCoreWorker.exe, usoclient.exe, TrustedInstaller.exe, wuauclt.exe | Windows Update | no |
| wmiprvse.exe | WMI request (e.g. Restart-Computer or remote management) | no |

**Startup.** The script waits (30 s by default), then tries to connect to the SMTP server several times (see Configuration). Then it evaluates the event log:

- **Clean shutdown** - the event before the current kernel start (Kernel-General 12) is a kernel stop (Kernel-General 13). The e-mail *Back online* contains the request from event 1074, the action type from event 109 (Kernel-Power), the time the shutdown completed and the downtime.
- **Unexpected restart** - the event before the current kernel start is another kernel start, the clean stop is missing. The start of the outage is estimated as the latest of: the last event before the boot in the System, Application and Security logs, and the last "alive" stamp stored in event 6008. Events 41 (Kernel-Power) and 6008 are listed when they are already in the log.

**Downtime** is measured from the shutdown request (event 1074) to the moment the startup notification starts running. For an unexpected restart it is marked as estimated.

The scheduled task has two triggers: *At startup* for a full boot, and event 27 (Kernel-Boot) with boot type 1 or 2 for fast startup or resume from hibernation, which do not fire *At startup*.

### Event log sources

All values are read from language independent fields wherever possible.

| Event | Used for | Notes |
|---|---|---|
| User32 1074 | Who, process, reason, comment, request time | `param5` (shutdown type) and `param3` (reason) are localized. English and Czech shutdown types are translated to Restart / Power off, other languages are shown as written. |
| Kernel-Power 109 | Restart or power off (`ShutdownActionType` 5 / 6) | Written after the shutdown script runs, used only at startup. |
| Kernel-General 12 / 13 | Kernel start and stop times, clean or unexpected shutdown | Provider must be filtered, Hyper-V-Netvsc also writes event ID 12. |
| Kernel-Boot 27 | Boot time and boot type | Trigger of the scheduled task for fast startup. |
| EventLog 6008 | Last alive stamp before an unexpected shutdown | Text values are localized, the time is read from the binary data. Written about 30 s after the boot. |
| Kernel-Power 41 | Listed in the Unexpected restart e-mail | Written a few seconds after the boot. |

## System requirements

- Windows Server 2022 or newer, or Windows 10 / 11
- Windows PowerShell 5.1
- Administrator rights for installation
- Anonymous SMTP relay reachable from the computer (default port 25)

## Installation

1. Copy the whole folder to the computer.
2. Create a configuration: copy `config\Send-RestartNotification-EXAMPLE.psd1` to `config\Send-RestartNotification-<name>.psd1` (for example `-EON`), set it up (SMTP server, sender, recipients, company name, texts) and replace the images in `config\img\` if needed. The package can hold several configurations.
3. Run `1Install-RestartNotification.cmd` as administrator and select the configuration.
4. Send test e-mails: `& "C:\Program Files\ZOBEC\RestartNotification\Send-RestartNotification.ps1" -CallAction Test`

### Interactive installation, reinstallation and uninstallation

Run the installer without parameters. When the product is not installed, it lists the configurations `config\Send-RestartNotification-<name>.psd1` (EXAMPLE is ignored) and installs the selected one. When the product is already installed, it shows the installed version and configuration and offers:

- **Reinstall** - select a configuration and install again, the existing configuration is replaced (a backup is kept),
- **Uninstall** - remove the product, see below,
- **Cancel**.

The installer is also copied to the installation folder. Run from there, it offers only uninstallation, reinstallation needs the installation package.

### Unattended installation

```powershell
.\1Install-RestartNotification.ps1 -ConfigName EON
```

Installs `config\Send-RestartNotification-EON.psd1` without any questions, for example when preparing a server image. An existing installation and configuration are always replaced. `EXAMPLE` is not accepted. Uninstallation is interactive only.

### What the installer does

- stops when no configuration is found or the selected one is not valid (checked by `-CallAction ValidateConfig` in a temporary copy, before anything is changed),
- copies the script, the installer, the documents and `img\setup.ico` to `%ProgramFiles%\ZOBEC\RestartNotification\`,
- copies the selected configuration as `config\Send-RestartNotification.psd1` together with `config\img\`; an existing configuration is backed up to `%ProgramData%\ZOBEC\RestartNotification\backup\` and replaced, other configuration files in the package are not copied,
- checks the installed configuration again and stops before registering anything when it is not valid,
- adds the script to the local Group Policy as a PowerShell shutdown script with parameters `-CallAction Shutdown`, keeps all other scripts and updates its own entry on reinstallation,
- stops when `psscripts.ini` is not consistent (duplicate keys, gaps in numbering, unknown lines); in that case set up the shutdown script manually in `gpedit.msc`,
- saves a backup of `psscripts.ini` and `gpt.ini` to `%ProgramData%\ZOBEC\RestartNotification\backup\`,
- runs `gpupdate /target:computer /force`,
- creates or replaces the scheduled task `\ZOBEC\Send Restart Notification` (runs as SYSTEM),
- adds the entry *ZOBEC Send Restart Notification* to Programs and Features (version, publisher, support link, installed configuration in Comments). The Uninstall, Change and Repair buttons are disabled; everything is done by the installer.

### Uninstallation

Run the installer from the package (`1Install-RestartNotification.cmd`) or from the installation folder (`powershell.exe -ExecutionPolicy Bypass -File "C:\Program Files\ZOBEC\RestartNotification\1Install-RestartNotification.ps1"`, the same command is shown as the uninstall string in Programs and Features) and select **Uninstall**. It removes the scheduled task, its own shutdown script entry from the local Group Policy (other scripts are kept, then `gpupdate`), the Programs and Features entry and the folder `%ProgramFiles%\ZOBEC\RestartNotification`. Logs and backups in `%ProgramData%\ZOBEC\RestartNotification` are kept.

The installer does not change any other setting of the computer. The scheduled task of the previous version (*Device was Restarted Notification*) is not removed.

## Configuration

In the package: `config\Send-RestartNotification-<name>.psd1`, one file per customer or environment, created from `config\Send-RestartNotification-EXAMPLE.psd1`. On the computer: `config\Send-RestartNotification.psd1` next to the installed script, a copy of the selected configuration. The file must be saved as UTF-8 with BOM. Comments in the file describe each value.

- `Smtp` - server, port, sender address, sender display name (optional) and recipients. Only anonymous relay is supported.
- `Smtp.UseTls` - STARTTLS encryption (`$true` / `$false`). Works with anonymous relay, the server must support STARTTLS and have a valid certificate.
- `Smtp.StartupDelaySeconds`, `Smtp.Attempts`, `Smtp.RetrySeconds` - connection attempts to the SMTP server before sending (30 s delay at startup, allowed 0-300 s, 12 attempts, 3 s apart). The delay applies to Startup only, Shutdown and Test try immediately.
- `Branding` - company name and logo row below the banner, notice text below the company name (`NoticeText`) and colors. The footer contains only information about the script, its background uses `Colors.Primary`.
- `Texts` - subject, status label, heading and introduction for each e-mail variant. Placeholders: `{Server}`, `{Action}`, `{InitiatedBy}`, `{Downtime}`, `{Company}`.

Images have fixed names in `config\img\`. A missing image is left out of the e-mail. Images are embedded in the message (inline attachments), so they are shown without downloading anything.

## Usage

```powershell
.\Send-RestartNotification.ps1 -CallAction Test       # three sample e-mails, subject starts with [TEST]
.\Send-RestartNotification.ps1 -CallAction ValidateConfig   # checks the configuration only
.\Send-RestartNotification.ps1 -CallAction Shutdown   # used by the shutdown script
.\Send-RestartNotification.ps1 -CallAction Startup    # used by the scheduled task
```

### Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Error, notification was not sent |
| 2 | Notification sent with warnings (see the log) |

## Logs

`%ProgramData%\ZOBEC\RestartNotification\Send-RestartNotification-log-YYYYMMDD-hhmmss.txt`, one file per run. Files older than 120 days are deleted automatically.

## Known limitations

- The shutdown script does not run after a crash, a power loss or a forced reset from the hypervisor. Only the *Unexpected restart* e-mail is sent after the next start.
- Fast startup (Windows 10 / 11) and resume from hibernation have not been verified on a physical machine yet. In Hyper-V virtual machines fast startup does not take effect.
- A full boot after a fast startup shutdown (Windows 10 / 11) may be reported as an unexpected restart, because the hybrid shutdown probably does not write Kernel-General 13. Not verified yet.
- Run the test after installation to verify that the PowerShell execution policy of the computer allows the Group Policy shutdown script and that the SMTP relay accepts messages from the computer.
- Azure blocks outbound TCP port 25 for most subscription types.
