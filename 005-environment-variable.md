# Environment Variable

```powershell
# ================================================================
# devtool Environment Variables - Fix & Cleanup (v4)
# Run as Administrator.
# Idempotent: safe to run multiple times.
# Key fix: correct fnm argument order (--shell powershell --resolve-engines)
#          and remove any previously written WRONG fnm init block.
# ================================================================

$DEVTOOL = "C:\Hub\devtool"

# ---------- 0. Admin check ----------
$isAdmin = ([Security.Principal.WindowsPrincipal] `
    [Security.Principal.WindowsIdentity]::GetCurrent()
).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

if (-not $isAdmin) {
    Write-Warning "Not running as Administrator. Machine PATH writes will fail."
    Write-Warning "Right-click PowerShell -> Run as administrator, then re-run."
}

# ---------- Helper: safe append ----------
function Add-ToPath {
    param(
        [Parameter(Mandatory)][string]$Entry,
        [Parameter(Mandatory)][ValidateSet("User","Machine")][string]$Scope
    )
    $current = [Environment]::GetEnvironmentVariable("Path", $Scope)
    if ([string]::IsNullOrEmpty($current)) { $current = "" }

    $parts = $current -split ';' | Where-Object { $_ -and $_.Trim() -ne '' }

    if ($parts -contains $Entry) {
        Write-Host "[SKIP] Already present: $Entry  ($Scope)"
        return
    }
    $new = ($parts + $Entry) -join ';'
    [Environment]::SetEnvironmentVariable("Path", $new, $Scope)
    Write-Host "[ OK ] Added: $Entry  ($Scope)"
}

# ---------- 1. Execution policy ----------
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
Write-Host "[ OK ] ExecutionPolicy = RemoteSigned (CurrentUser)"

# ---------- 2. DEVTOOL root ----------
[Environment]::SetEnvironmentVariable("DEVTOOL", $DEVTOOL, "User")
Write-Host "[ OK ] DEVTOOL = $DEVTOOL  (User)"

# ---------- 3. All binaries -> MACHINE PATH ----------
Add-ToPath -Entry "$DEVTOOL\git\cmd" -Scope Machine
Add-ToPath -Entry "$DEVTOOL\git\bin" -Scope Machine
Add-ToPath -Entry "$DEVTOOL\fnm"     -Scope Machine
Add-ToPath -Entry "$DEVTOOL\uv"      -Scope Machine

# ---------- 4. fnm environment variables ----------
[Environment]::SetEnvironmentVariable("FNM_DIR",              "$DEVTOOL\fnm\node-versions",         "User")
[Environment]::SetEnvironmentVariable("FNM_NODE_DIST_MIRROR", "https://npmmirror.com/mirrors/node", "User")
Write-Host "[ OK ] FNM_DIR / FNM_NODE_DIST_MIRROR  (User)"

# ---------- 5. Profile: rebuild fnm init block (removes old wrong version) ----------
if (!(Test-Path -Path $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force | Out-Null
    Write-Host "[ OK ] Created Profile: $PROFILE"
} else {
    Write-Host "[INFO] Profile exists: $PROFILE"
}

# --- Read profile, strip any previous fnm init block (correct or wrong) ---
$raw = Get-Content -Path $PROFILE -Raw -ErrorAction SilentlyContinue
if ([string]::IsNullOrEmpty($raw)) { $raw = "" }

# Remove any block between "# ---- fnm init" and "# ---- end fnm init ----"
$cleaned = [regex]::Replace(
    $raw,
    '(?ms)^\s*# ---- fnm init.*?# ---- end fnm init ----\s*\r?\n?',
    ''
)

# Also remove the older marker variant (added by setup-devtool.ps1)
$cleaned = [regex]::Replace(
    $cleaned,
    '(?ms)^\s*# ---- fnm init \(added by setup-devtool\.ps1\).*?# ---- end fnm init ----\s*\r?\n?',
    ''
)

# Trim trailing blank lines
$cleaned = $cleaned.TrimEnd() + "`r`n"

# --- Append the corrected fnm init block ---
$fnmBlock = @'

# ---- fnm init ----
# NOTE: argument order matters.
#   --shell powershell        : emit PowerShell script
#   --resolve-engines         : boolean flag, MUST NOT be followed by a value
#   --use-on-cd               : auto-switch on directory change
if (Get-Command fnm -ErrorAction SilentlyContinue) {
    fnm env --use-on-cd --shell powershell --resolve-engines |
        Out-String | Invoke-Expression
}
# ---- end fnm init ----
'@

$cleaned = $cleaned.TrimEnd() + "`r`n" + $fnmBlock
Set-Content -Path $PROFILE -Value $cleaned -Encoding UTF8
Write-Host "[ OK ] Profile fnm init block rewritten (correct argument order)"

# ---------- 6. Broadcast environment change ----------
Add-Type -Namespace Win32 -Name NativeMethods -MemberDefinition @"
[DllImport("user32.dll", SetLastError = true, CharSet = CharSet.Auto)]
public static extern IntPtr SendMessageTimeout(
    IntPtr hWnd, uint Msg, UIntPtr wParam, string lParam,
    uint fuFlags, uint uTimeout, out UIntPtr lpdwResult);
"@ -ErrorAction SilentlyContinue

[Win32.NativeMethods]::SendMessageTimeout(
    [IntPtr]0xffff, 0x1a, [UIntPtr]::Zero, "Environment", 2, 5000,
    [ref]([UIntPtr]::Zero)) | Out-Null

# ---------- 7. Refresh current session ----------
$machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
$userPath    = [Environment]::GetEnvironmentVariable("Path", "User")
$env:Path    = ($machinePath, $userPath | Where-Object { $_ }) -join ';'
Write-Host "[ OK ] Current session PATH refreshed"

# ---------- 8. Initialize fnm in THIS session ----------
if (Get-Command fnm -ErrorAction SilentlyContinue) {
    fnm env --use-on-cd --shell powershell --resolve-engines |
        Out-String | Invoke-Expression
    Write-Host "[ OK ] fnm environment injected into current session"
} else {
    Write-Warning "fnm not found in PATH. Cannot inject fnm environment."
}

# ---------- 9. Verification ----------
Write-Host ""
Write-Host "---------- Verification ----------"

$checks = @(
    @{ Name = "DEVTOOL";              Value = $env:DEVTOOL },
    @{ Name = "FNM_DIR";              Value = $env:FNM_DIR },
    @{ Name = "FNM_MULTISHELL_PATH";  Value = $env:FNM_MULTISHELL_PATH },
    @{ Name = "git";                  Cmd   = "git" },
    @{ Name = "fnm";                  Cmd   = "fnm" },
    @{ Name = "uv";                   Cmd   = "uv"  }
)

foreach ($c in $checks) {
    if ($c.Cmd) {
        $found = Get-Command $c.Cmd -ErrorAction SilentlyContinue
        if ($found) { Write-Host ("[PASS] {0,-20} -> {1}" -f $c.Cmd, $found.Source) }
        else        { Write-Host ("[FAIL] {0,-20} -> not found in PATH" -f $c.Cmd) }
    } else {
        if ($c.Value) { Write-Host ("[PASS] {0,-20} = {1}" -f $c.Name, $c.Value) }
        else          { Write-Host ("[FAIL] {0,-20} = <empty>" -f $c.Name) }
    }
}

Write-Host ""
Write-Host "=========================================="
Write-Host "Done. Current session is ready."
Write-Host "Other running processes need a restart."
Write-Host "=========================================="
```