---
name: terminal-safe-write
description: "Create or edit files through terminal commands when ordinary editor, patch, or API writes may encrypt or corrupt files. Use when the user asks to avoid encryption, asks for terminal-created files, or needs .md/.py files created with PowerShell New-Item and Set-Content."
---

# Terminal Safe Write

Use this skill when the user says files written by normal editor, patch, or API paths may be encrypted, while files created and written through the terminal remain readable.

## Core Rule

Make the terminal process perform both operations:

1. Create the file with `New-Item -ItemType File -Path <target> -Force`.
2. Write the file with `Set-Content -Path <target> -Encoding UTF8`.

Do not use `apply_patch`, editor save, Python direct file writes, or other non-terminal write paths for protected files.

## Workflow

1. Resolve the intended directory first, for example `Resolve-Path -LiteralPath .\file2md`.
2. Create needed folders with `New-Item -ItemType Directory -Path <dir> -Force`.
3. Create the target file with `New-Item -ItemType File -Path <file> -Force | Out-Null`.
4. Pipe literal content from the terminal into `Set-Content -Path <file> -Encoding UTF8`.
5. If the user restricts file extensions, enforce the suffix before writing. For example, refuse non-`.md` outputs when the user says only markdown is allowed.
6. For Python scripts that generate files, have Python call `powershell.exe` and pass the target path through an environment variable. The generated file should still be created and written by PowerShell, not by Python file APIs.
7. Avoid validation commands that create extra files, such as `python -m py_compile`, unless the user explicitly allows them. If a temporary cache is accidentally created, remove it after resolving the path and confirming it is inside the workspace.

## Python-to-PowerShell Pattern

Use this pattern when a Python script must create a markdown output without directly writing the target file:

```python
env = os.environ.copy()
env["TERMINAL_MD_TARGET"] = str(output_path)
command = [
    "powershell.exe",
    "-NoProfile",
    "-ExecutionPolicy",
    "Bypass",
    "-Command",
    (
        "$target = $env:TERMINAL_MD_TARGET; "
        "New-Item -ItemType File -Path $target -Force | Out-Null; "
        "$input | Set-Content -Path $target -Encoding UTF8"
    ),
]
subprocess.run(command, input=content, text=True, check=True, env=env)
```

## Validation

After writing, verify through terminal reads:

- Use `Get-Content -LiteralPath <file> -TotalCount 20` to confirm readable content.
- Use `Get-ChildItem -Force -LiteralPath <dir>` to confirm only expected files were created.
- Report any accidental auxiliary files and clean them only after checking the path is inside the workspace.

## Reference

For ready-to-copy terminal templates and the rationale, read `references/terminal-safe-write.md`.