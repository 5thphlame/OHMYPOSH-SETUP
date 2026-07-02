# Customizing Windows Terminal with Oh My Posh: A Real Troubleshooting Journey

If you've ever admired those slick, information-rich terminal prompts developers show off online — git branch status, execution time, OS icons, all in a clean segmented bar — and wondered how to get that on Windows without diving into a full Linux setup, this post is for you.

I recently went through this exact process on a fresh Windows 11 machine. It wasn't a clean, five-minute install — I hit real errors along the way, including execution policy blocks, missing themes, and a failed PowerToys install. I'm documenting the whole thing, warts and all, because the troubleshooting is often more useful than the happy path.

## The Goal

I wanted a Powerlevel10k-style prompt — the kind popular in the Zsh/Linux world — but without needing WSL or a Linux shell running underneath. That meant going with **Oh My Posh**, a native Windows (and cross-platform) prompt theming engine that works directly with PowerShell.

## Step 1: Choosing a Terminal

Before any theming, it's worth clarifying a common point of confusion: **Command Prompt and Windows Terminal are not the same thing.**

- **Command Prompt (`cmd.exe`)** is a *shell* — the actual program that interprets your commands.
- **Windows Terminal** is a *terminal application* — the window/UI that hosts shells. It can run Command Prompt, PowerShell, WSL distros, or Azure Cloud Shell, all as tabs in one app.

Think of the shell as the engine and the terminal as the car. Windows Terminal is the modern, GPU-accelerated, tabbed "car" that Microsoft now ships for all of this, and it's the natural home for any prompt customization.

## Step 2: Installing Oh My Posh

With PowerShell open:

```powershell
winget install JanDeDobbeleer.OhMyPosh -s winget
```

Close and reopen the terminal afterward so `PATH` updates take effect.

## Step 3: Installing a Nerd Font

Themed prompts rely on special glyphs (git icons, OS logos, etc.) that regular fonts don't include. Oh My Posh can install one for you:

```powershell
oh-my-posh font install meslo
```

Then, in Windows Terminal: **Settings → your PowerShell profile → Appearance → Font face → `MesloLGS NF`**.

## Step 4: Wiring Oh My Posh into Your PowerShell Profile

This is where the real troubleshooting began.

First, I checked whether a PowerShell profile file even existed:

```powershell
Test-Path $PROFILE
```

Since it didn't, I created one:

```powershell
New-Item -Path $PROFILE -Type File -Force
notepad $PROFILE
```

And added the initialization line:

```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" | Invoke-Expression
```

### Error #1: Script Execution Disabled

Reloading the profile threw this:

```
File ...\Microsoft.PowerShell_profile.ps1 cannot be loaded because running scripts is disabled on this system.
```

Windows disables script execution by default for security. The fix is to loosen the policy — just for your user account, not machine-wide:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

`RemoteSigned` lets locally-written scripts (like your own `$PROFILE`) run freely, while still requiring a digital signature for anything downloaded from the internet. It's the standard setting most developers use — not a security free-for-all.

If you're on a locked-down corporate machine and this doesn't take effect, check for policy overrides with:

```powershell
Get-ExecutionPolicy -List
```

A `MachinePolicy` or `UserPolicy` entry set by IT will override anything you do at the user level.

### Error #2: `CONFIG NOT FOUND`

After fixing the execution policy, the profile loaded — but Oh My Posh itself complained:

```
CONFIG NOT FOUND
```

This error comes directly from Oh My Posh, and it means the `--config` path you gave it doesn't point to a real file. I worked through this methodically:

**Check the environment variable it depends on:**

```powershell
echo $env:POSH_THEMES_PATH
```

It came back **empty**. That was the smoking gun — `POSH_THEMES_PATH` is supposed to be set automatically during installation, but it hadn't been.

**Confirm Oh My Posh itself was actually installed correctly:**

```powershell
oh-my-posh version
Get-Command oh-my-posh
```

This showed version `29.18.0`, installed via the Windows Store-linked path (`...\WindowsApps\oh-my-posh.exe`). That particular install method doesn't bundle a themes folder — themes have to be fetched separately.

**Searching the whole disk for the theme file confirmed it:**

```powershell
Get-ChildItem -Path C:\ -Filter "jandedobbeleer.omp.json" -Recurse -ErrorAction SilentlyContinue -Force
```

Nothing came back. The file genuinely didn't exist anywhere on the machine.

### The Fix: Manually Downloading the Themes

Since the installer hadn't provided them, I downloaded the themes package directly from the Oh My Posh GitHub releases, matched to my installed version:

```powershell
# 1. Create a folder to hold the themes
New-Item -Path "$env:LOCALAPPDATA\Programs\oh-my-posh\themes" -ItemType Directory -Force

# 2. Download the themes zip for the matching version
Invoke-WebRequest -Uri "https://github.com/JanDeDobbeleer/oh-my-posh/releases/download/v29.18.0/themes.zip" -OutFile "$env:TEMP\themes.zip"

# 3. Extract it
Expand-Archive -Path "$env:TEMP\themes.zip" -DestinationPath "$env:LOCALAPPDATA\Programs\oh-my-posh\themes" -Force

# 4. Set the environment variable permanently (not just for this session)
[System.Environment]::SetEnvironmentVariable('POSH_THEMES_PATH', "$env:LOCALAPPDATA\Programs\oh-my-posh\themes", 'User')
```

**Important:** environment variables set this way only apply to *new* terminal sessions. I had to fully close and reopen Windows Terminal before it took effect.

After restarting:

```powershell
echo $env:POSH_THEMES_PATH
# C:\Users\PAmah\AppData\Local\Programs\oh-my-posh\themes
```

Reloading the profile finally worked, and the prompt came alive — showing the shell name, execution time per command, and a clean segmented layout, all rendered correctly thanks to the Nerd Font.

## Step 5: Browsing Themes

Oh My Posh ships dozens of built-in themes. Once the themes folder actually existed, I could preview all of them at once:

```powershell
Get-ChildItem "$env:POSH_THEMES_PATH" -Filter '*.omp.json' | ForEach-Object {
    Write-Host "Theme: $($_.BaseName)" -ForegroundColor Cyan
    oh-my-posh print primary --config $_.FullName
    Write-Host ""
}
```

Worth trying: `jandedobbeleer` (the default, and what I ended up using), `atomic`, `powerlevel10k_rainbow` (closest match to the classic P10k look), `paradox`, and `night-owl`.

To switch permanently, edit `$PROFILE` and change the theme filename in the `--config` path, then reload with `. $PROFILE`.

## Step 6: Extra Polish

A couple of optional add-ons rounded things out:

**Terminal-Icons** — adds file/folder icons to `ls`/`dir` output:

```powershell
Install-Module -Name Terminal-Icons -Scope CurrentUser -Repository PSGallery
```

*(Note: if PSGallery is untrusted, you'll be prompted to confirm — answer `Y` to proceed.)*

Then add to `$PROFILE`:

```powershell
Import-Module -Name Terminal-Icons
```

**PSReadLine** — improves command-line editing and history:

```powershell
Install-Module -Name PSReadLine -Scope CurrentUser -Force
```

**PowerToys** — a broader Windows productivity toolset (window management, color picker, etc.), unrelated to the terminal itself but a common companion install. This one actually failed for me with an MSI error (`0x80070643`):

```
Installer failed with exit code: 1603
```

The install log pointed to a generic Windows Installer failure rather than a download or permissions issue. The fix was running the installer manually, with elevation, instead of relying on winget's silent install:

1. Download the latest release manually from the [PowerToys GitHub releases page](https://github.com/microsoft/PowerToys/releases/latest)
2. Right-click the `.exe` → **Run as administrator**

This is a good reminder that `winget`'s silent install flag can trip over permission issues that a normal interactive installer handles without complaint.

## Final Result

What started as a single `winget install` command turned into a proper debugging session — execution policies, missing environment variables, and an installer that silently skipped a required asset. But each error had a clear, traceable cause, and the end result is a fully themed, icon-rich PowerShell prompt running natively in Windows Terminal, no WSL or Linux layer required.

**Quick recap of the core fix sequence, if you hit the same wall:**

1. `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` — if scripts won't run
2. Check `$env:POSH_THEMES_PATH` — if it's empty, that's your `CONFIG NOT FOUND` culprit
3. Manually download `themes.zip` from the Oh My Posh GitHub releases matching your installed version
4. Set `POSH_THEMES_PATH` permanently with `[System.Environment]::SetEnvironmentVariable(...)`
5. **Fully restart your terminal** — env variables won't apply to already-open sessions

If you're setting this up yourself, budget a bit more time than the "five-minute install" tutorials suggest — but every one of these issues has a clean fix once you know where to look.
