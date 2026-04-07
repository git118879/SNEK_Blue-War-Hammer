# Blue Hammer Vulnerability Documentation & Reimplementation - SNEK Blue War Hammer

Great thanks to [Nightmare-Eclipse](https://github.com/Nightmare-Eclipse/BlueHammer) for making this exploit POC public.

## Overview

`SNEK_BlueWarHammer.cpp` is the primary source file for the BlueHammer proof-of-concept. It is designed to exploit Windows Defender update behavior and use the Defender update mechanism to access protected files, culminating in privilege escalation through pass-the-hash techniques.

The main workflow is:

1. Detect a new Windows Defender signature update.
2. Download the update package from Microsoft's update URL.
3. Extract the update cabinet and store the update payload in memory.
4. Trigger Defender through RPC to force it to write the update payload.
5. Observe the Defender update directory creation.
6. Use a symbolic link / VSS-based technique to access protected files.
7. Copy the leaked protected file to a user-accessible location.
8. Parse the leaked SAM hive to extract NTLM hashes.
9. Perform privilege escalation using pass-the-hash techniques to spawn an elevated shell.

## Dependencies and Runtime Requirements

This project uses a mixture of Win32 APIs, COM, Windows Update Agent interfaces, native NT APIs, RPC stubs generated from `windefend.idl`, and additional libraries for pass-the-hash functionality.

Key dependencies:

- `wininet.lib`
- `ktmw32.lib`
- `Shlwapi.lib`
- `Rpcrt4.lib`
- `ntdll.lib`
- `Cabinet.lib`
- `Wuguid.lib`
- `CldApi.lib`
- `userenv.lib`
- `Secur32.lib`
- `wbemuuid.lib`

The code also imports `windefend_h.h` and `offreg.h` for RPC and registry access.

## Structure

The source file is divided into the following major sections:

- Top-level imports, macro definitions, and runtime-loaded NT imports.
- RPC allocation helpers required for MIDL-generated code.
- Defender RPC caller logic.
- Cabinet extraction and update file retrieval functions.
- Windows Update scanning and update detection logic.
- VSS and Defender trigger logic.
- SAM hive parsing and NTLM hash extraction.
- Pass-the-hash privilege escalation and shell spawn.
- The main application entry point.

## Native NT Imports

The code obtains NT native functions from `ntdll.dll` at runtime using `GetProcAddress`.
These functions are not part of the standard Win32 API and are used for low-level operations such as:

- `NtCreateSymbolicLinkObject`
- `NtOpenDirectoryObject`
- `NtQueryDirectoryObject`
- `NtSetInformationFile`

This runtime resolution allows the exploit to interact with low-level kernel objects and reparse points without static dependencies.

## RPC Allocation Helpers

The functions `midl_user_allocate` and `midl_user_free` are required by the MIDL-generated RPC stubs.
They provide the client runtime with malloc/free callbacks for marshaling and unmarshaling RPC parameters.

## Defender RPC Call Flow

### `CallWD`

`CallWD` establishes an RPC binding to the local Defender service and invokes `ServerMpUpdateEngineSignature`.

- It builds an RPC string binding using the Defender interface UUID.
- It creates a binding handle from that string.
- It calls the Defender engine update RPC method.
- It signals a completion event when the call is finished.

The call is executed on a dedicated worker thread so the main code can continue monitoring file system events concurrently.

### `WDCallerThread`

`WDCallerThread` is a thin wrapper that turns the call into a thread-compatible entry point.
It validates the input pointer and calls `CallWD`.

## Defender Update Extraction

### `GetUpdateFiles`

`GetUpdateFiles` downloads the latest Defender update package directly from Microsoft and extracts it from the CAB format.

The function does the following:

- Creates an `InternetOpen` session.
- Opens the Defender update URL.
- Queries the HTTP content length to size the download buffer.
- Reads the entire update package into memory.
- Locates the embedded CAB file inside the downloaded package.
- Uses FDI (Microsoft Cabinet API) callbacks to extract the CAB payload in memory.
- Returns a linked list of `UpdateFiles` structures containing file names, buffers, and sizes.

This function is central to the exploit because it provides the raw update files that are later used to trigger Defender behavior.

## Update Scan Logic

### `CheckForWDUpdates`

`CheckForWDUpdates` uses the Windows Update Agent COM interfaces to determine whether a new Defender update is available.

The function performs the following steps:

- Initializes COM.
- Creates an `IUpdateSession` object.
- Creates an `IUpdateSearcher` object.
- Performs a search for available updates.
- Inspects each returned update for Defender-related metadata.
- Returns success when a suitable update is found.

If COM initialization or any update interface call fails, the function sets `*criterr` to true and reports failure.

## VSS Trigger Logic

### `TriggerWDForVS`

`TriggerWDForVS` attempts to force Defender to create a volume shadow copy path.

It does so by:

- Generating a unique temporary directory under `%TEMP%`.
- Writing a specially crafted file into that directory.
- Creating a worker thread to monitor shadow copy/VSS state.
- Waiting for Defender to access the file and create a new VSS-ready location.

The function returns `true` only when Defender has been successfully coerced into producing the expected shadow copy path.

## SAM Hive Parsing and Hash Extraction

### SAM Database Access

After leaking the SAM hive, the code uses offline registry access (`offreg.lib`) to parse the SAM database:

- Extracts LSA secret keys for decryption.
- Decrypts password encryption keys.
- Parses user accounts to extract NTLM hashes.
- Filters out invalid or system accounts (e.g., WDAGUtilityAccount).

This provides the raw NTLM hashes needed for pass-the-hash attacks.

## Pass-the-Hash Privilege Escalation

### Overview

The exploit implements multiple pass-the-hash techniques to achieve true privilege escalation:

1. **Token Elevation**: Uses LogonUserEx with reset passwords to obtain elevated tokens.
2. **Sys Shell**: Creates a service to run as SYSTEM and spawn a shell.
3. **Pth Elevation**: Leverages the extracted NTLM hash directly for authentication.

### `SpawnShellWithPassTheHash`

This function attempts to use the NTLM hash for elevated process creation:

- **WMI/COM Elevation**: Uses Windows Management Instrumentation with NTLM authentication to spawn processes.
- **SSPI/NTLM Authentication**: Implements Security Support Provider Interface for hash-based authentication.
- **Environment Block Creation**: Ensures proper environment setup for impersonated processes.

### Shell Spawn Logic

The shell spawn attempts multiple fallback mechanisms:

1. **CreateProcessAsUserW** with environment block (fixes DLL\_INIT\_FAILED errors).
2. **CreateProcessWithLogonW** with profile loading.
3. **Pass-the-Hash via WMI** for true hash-based elevation.

## Local System and Session Management

### `IsRunningAsLocalSystem`

Determines whether the current process token corresponds to the Local System account.
This is useful because the tool can behave differently when executed as SYSTEM.

### `LaunchConsoleInSessionId`

If the tool is running as SYSTEM and receives a session ID argument,
`LaunchConsoleInSessionId` duplicates the current token, sets its session ID,
and spawns `conhost.exe` inside that session.

This allows the exploit to open an interactive console in the context of another desktop session.

## Critical Path

The `wmain` function is the orchestrator for the exploit routine.

Key behavior:

- If the process is running as SYSTEM and a valid session ID is provided, it launches a console in that session and exits.
- Otherwise, it enters the Defender update exploitation workflow.
- The workflow polls for a valid Defender signature update.
- When an update becomes available, it downloads and extracts the update files.
- It triggers Defender to operate on the extracted update files and create a VSS path.
- It repeatedly attempts to leak protected system files through the Defender update directory.
- After successfully leaking a file, it hands the file off to the post-exploit shell-spawning helper.
- Cleanup code releases handles, closes events, and frees allocated memory.

## Important Notes

- The project is not intended for benign production use; it is a proof-of-concept exploit.
- The implementation relies on Defender internals and Windows Update behavior that may change between versions.
- The code assumes it can run with high privileges and may fail with insufficient rights.
- Memory and handle cleanup is performed in a `cleanup` block using `goto` to centralize resource release.
- Pth techniques require valid NTLM hashes and may be affected by modern security mitigations.

## Code Comments

The source file includes documentation in the form of comments for all major sections and important control points:

- Top-level purpose and runtime imports.
- Native NT export loading.
- RPC runtime helper callbacks.
- Defender RPC invocation.
- Update download and extraction.
- Windows Update polling.
- Shadow copy / VSS trigger.
- SAM parsing and hash extraction.
- Pass-the-hash implementation details.
- Session console launching.
- Error handling and fallback mechanisms.

## Build Instructions

To build the project:

1. Open `SNEK_BlueWarHammer.sln` in Visual Studio 2022.
2. Ensure vcpkg is installed and configured for x64-windows triplet.
3. Build the Release|x64 configuration.
4. The executable will be generated in `x64\Release\SNEK_BlueWarHammer.exe`.

## Usage

Run the exploit with logging:

```
SNEK_BlueWarHammer.exe --log-steps
```

This will execute the full exploit chain and display progress markers.

## Security Considerations

This tool demonstrates a critical vulnerability in Windows Defender. Use only in controlled environments for research purposes. The exploit may be detected by security software and should not be used maliciously.

This documentation is intended to explain every major part of `SNEK_BlueWarHammer.cpp` in a professional manner.

### Provided by: ATroubledSnake & The SNEK Initiative. Long live freeware.
