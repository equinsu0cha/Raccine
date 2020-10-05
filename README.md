![Raccine](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/raccine_logo.png)

# Raccine

A Simple Ransomware Protection 

## Why

We see ransomware delete all shadow copies using `vssadmin` pretty often. What if we could just intercept that request and kill the invoking process? Let's try to create a simple vaccine.

![Ransomware Process Tree](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen4.png)

## How it works

We [register a debugger](https://attack.mitre.org/techniques/T1546/012/) for `vssadmin.exe` (and `wmic.exe`), which is our compiled `raccine.exe`. Raccine is a binary, that first collects all PIDs of the parent processes and then tries to kill all parent processes. 

Avantages:

- The method is rather generic
- We don't have to replace a system file (`vssadmin.exe` or `wmic.exe`), which could lead to integrity problems and could break our raccination on each patch day 
- The changes are easy to undo
- Should work on all Windows versions from Windows 2000 onwards
- No running executable or additional service required (agent-less)

Disadvantages / Blind Spots:

- The legitimate use of `vssadmin.exe delete shadows` (or any other blacklisted combination) isn't possible anymore
- It even kills the processes that tried to invoke `vssadmin.exe delete shadows`, which could be a backup process
- This won't catch methods in which the malicious process isn't one of the processes in the tree that has invoked `vssadmin.exe` (e.g. via `schtasks`)

### The Process

1. Invocation of `vssadmin.exe` (and `wmic.exe`) gets intercepted and passed to `raccine.exe` as debugger (`vssadmin.exe delete shadows` becomes `raccine.xe vssadmin.exe delete shadows`)
2. We then process the command line arguments and look for malicious combinations. 
3. If no malicious combination could be found, we create a new process with the original command line parameters. 
4. If a malicious combination could be found, we collect all PIDs of parent processes and the start killing them (this should be the malware processes as shown in the screenshots above). Raccine shows a command line window with the killed PIDs for 5 seconds and then exits itself. 

Malicious combinations:

- `delete` and `shadows` (vssadmin)
- `resize` and `shadowstorage` (vssadmin)
- `delete` and `shadowcopy` (wmic)

## Warning !!!

USE IT AT YOUR OWN RISK!

You won't be able to run commands that use the blacklisted commands on a raccinated machine anymore until your apply the uninstall patch `raccine-reg-patch-uninstall.reg`. This could break various backup solutions that run that specific command during their work. It will not only block that request but kills all processes in that tree including the backup solution and its invoking process.

If you have a solid security monitoring that logs all process executions, you could check your logs to see if `vssadmin.exe delete shadows` or `vssadmin.exe resize shadowstorage ...` is frequently or sporadically used for legitimate purposes in which case you should refrain from using Raccine.

## Version History

- 0.1.0 - Initial version that intercepted & blocked all vssadmin.exe executions
- 0.2.0 - Version that blocks only vssadmin.exe executions that contain `delete` and `shadows` in their command line and otherwise pass all parameters to a new process that invokes vssadmin with its original parameters
- 0.2.1 - Removed `explorer.exe` from the whitelist
- 0.3.0 - Supports the `wmic` method calling `delete shadowcopy`, no outputs for whitelisted process starts (avoids problems with wmic output processing)
- 0.4.0 - Supports logging to the Windows Eventlog for each blocked attempt, looks for more malicious parameter combinations

## Installation

1. Apply Registry Patch `raccine-reg-patch-vssadmin.reg` to intercept invocations of `vssadmin.exe`
2. Place `Raccine.exe` from the [release section](https://github.com/Neo23x0/Raccine/releases/) in the `PATH`, e.g. into `C:\Windows`

(For i386 architecture systems use `Raccine_x86.exe` and rename it to `Raccine.exe`)

### Eventlog Addon (Optional)

3. Run `InstallDll.bat` to install a message DLL that formats the eventlog entries generated by Raccine

This step is optional and purely cosmetic. Registering the DLL allows for well-formatted eventlog entries

Eventlog entry before DLL installation:

![Eventlog Entry Without DLL](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/eventlog-withoutdll.png)

Eventlog entry after DLL installation:

![Eventlog Entry With DLL](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/eventlog-withdll.png)

### Wmic Addon (Optional)

About 10-30% of Ransomware samples use `wmic` to delete the local shadowcopies. However, `wmic` is used for administrative activity far more often than `vssadmin`. The output of wmic often gets processed by automated scripts. It is unknown how a proxied execution through Raccine affects these scripts and programs. We've removed all outputs for cases in which no malicious parameter combination gets detected, but who knows?

4. Apply the `raccine-reg-patch-wmic.reg` patch to intercept invocations of `wmic.exe`

![Kill Run](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen5.png)

## Uninstall 

1. Run `raccine-reg-patch-uninstall.reg` 
2. Remove `Raccine.exe` (optional)

## Screenshot

Run `raccine.exe` and watch the parent process tree die (screenshot of v0.1)

![Kill Run](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen1.png)

## Pivot

In case that the Ransomware that your're currently handling uses a certain process name, e.g. `taskdl.exe`, you could just change the `.reg` patch to intercept calls to that name and let Raccine kill all parent processes of the invoking process tree.

## Help Wanted

I'd like to extend Raccine but lack the C++ coding skills, especially o the Windows platform.

### ~~1. Allow Certain Vssadmin Executions~~

***implemented by Ollie Whitehouse in v0.2.0***

Since Raccine is registered as a debugger for `vssadmin.exe` the actual command line that starts raccine.exe looks like

```bash
raccine.exe vssadmin.exe ... [params]
```

![raccine as debugger](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen3.png)

If we were able to process the command line options and apply filters to them, we could provide the following features: 

- Only block the execution in cases in which the parameters contains `delete shadows`
- Allow all other executions by passing the original parameters to a newly created process of `vssadmin.exe` (transparent pass-through)

### 2. Whitelist Certain Parents

We could provide a config file that contains white-listed parents for `vssadmin.exe`. If such a parent is detected, it would also pass the parameters to a new process and skip killing the process tree.

### 3. Create Shim Instead of Image File Execution Options Hack

The solution is outlined in this [tweet](https://twitter.com/cyb3rops/status/1312982510746374144?s=20) and related [talk](https://www.youtube.com/watch?v=LOsesi3QkXY&feature=youtu.be).

![raccine as debugger](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen-tweet1.png)

## FAQs

### Why did it even kill explorer.exe during its run?

Since malware tends to inject into `explorer.exe`, we thought it would be a good idea to kill even `explorer.exe` in order to avoid malicious code performing other operations on the system. What happens in real world examples is that a user that executed the Ransomware process would loose its windows task bar and desktop, while other programs like Microsoft Word or Outlook would still be running and the user would be able to save his work and close the respective programs before calling the helpdesk or simpy reboot the system. An expericend user could bring up task manager using `CTRL+ALT+Del` and start a new `explorer.exe` or just log off.

![raccine as debugger](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen-explorer-injection.png)

## Other Info

The right pronounciation is "Rax-Een".

## Important

Although it uses the term vaccine, I'd like to point out that the process applied by Raccine does not reflect the way vaccines work in medical science. Although its application introduces some changes to the system and doesn't require running processes like agents or scanners, it performs some destructive actions after being invoked by a malicious process. Vaccines in medical science don't work that way. Some members of the community asked me to clarify that and I am happy to comply with that request.

## Credits

- Florian Roth [@cyb3rops](https://twitter.com/cyb3rops)
- Ollie Whitehouse [@ollieatnccgroup](https://twitter.com/ollieatnccgroup)
